---
title: 记一次生产多线程数据导出bug排查
date: 2019-07-20 15:24:42
tags: [多线程, 并发, dubbo]
---
### 事情由来:
同事在写一个数据导出的需求的时候遇到个问题，调用dubbo后没响应，排查后是dubbo数据量多无法传输，
于是我用idea的evaluation动态调整了数据量的大小去运行，发现大概要到500条数据左右的时候consumer才能
得到响应，后来同事便调整了一下数据量，使用了线程池和callable,future来导出。于是bug开始产生了.  

### 发展：
同事代码写完后，数据接口调用的时间很长，得有二十来秒吧。由于需求来的**又快又急**，便匆忙上线了，结果第二天
使用的时候发现有的时候数据导出不完全正确，bug时有时无的。我开始还没有在意，直到他们一直在那里讨论，
而且这个需求已经耽误比较久了，于是想着反正我事情也做完了，我便看一下是什么鬼东西在作怪。

### 入手：
刚开始看代码的时候是这样的：各种奇怪的逻辑，就不贴上来的，贴上一段主要问题所在的那一段：
主要的逻辑是这样的：
先用查询参数查出数据量,
然后给每个线程指定一定的分页页数,
查询后,通过future的get方法拿出来放入到结果中......

~~~ java
public class controller{
    
    @RequestMapping(value = "/exportExcel")
	@ResponseBody
	public synchronized String  exportExcel(QueryParam queryParam, HttpServletResponse response) {
		queryParam.setIsExport("export");
		PageBo<Map<String,Object>> pageBoPage;
		List<Map<String,Object>> resultList = new ArrayList<>();
		// 根据数据量控制导出情况
		int totalData = excelService.countOfficialMicroBlogPutList(queryParam).getCount();
		int rows = 100;
		// 线程的个数
		int nThread = 5;
		//每个线程需要处理的页数
		int pageEachThread = pageTotal/nThread + 1;
		logger.info("每条线程需要查询的页数：[{}]",pageEachThread);
		if( pageEachThread < 1 ){
			for(int i = 0 ; i < pageTotal ; i++){
				param.setPage(i+1);
				param.setRows(rows);
				pageBoPage = excelService.getOfficialMicroBlogPutList(param);
				resultList.addAll(pageBoPage.getDataList());
			}
		} else {
			List<Future<List<Map<String, Object>>>> futures = new ArrayList<>();
			try{
				for(int j = 1; j <= nThread; j ++){
					int start = (1 + (j - 1) * pageEachThread);
					ThreadExcel worker = new ThreadExcel(start, rows, pageEachThread, param, excelService);
					Future<List<Map<String, Object>>> future = executorService.submit();
					futures.add(future);
				}
				for(Future<List<Map<String, Object>>> f : futures){
					resultList.addAll(f.get());
				}
			} catch (Exception e){
				logger.error("投放查询导出失败", e);
			}finally {
				executorService.shutdown();
			}
		}
		// ......其他代码
    }

    private class ThreadExcel implements Callable {
    
        private Integer start;
        private Integer rows;
        private Integer pageEachThread;
        private QueryParam queryParam;
        private ExcelService excelService;
    
        private ThreadExcel(Integer start,Integer rows,Integer pageEachThread,QueryParam queryParam, ExcelService excelService){
            this.start = start;
            this.rows = rows;
            this.pageEachThread = pageEachThread;
            this.queryParam = queryParam;
            this.excelService = excelService;
        }
    
        @Override
        public List<Map<String, Object>> call() {
            List<Map<String, Object>> list = new ArrayList<>();
            for (int k = start; k < start + pageEachThread; k++) {
                param.setPage(k);// --------- 这里注意一下
                param.setRows(rows);
                try{
                    PageBo<Map<String, Object>> pageList = excelService.getOfficialMicroBlogPutList(param);
                    list.addAll(pageList.getDataList());
                }catch (Exception e){
                    logger.error(e);
                }
            }
            return list;
        }
    }
}

~~~

额，应该是写多线程写的确实比较少吧，一拿到这块代码感觉改的地方至少有三点了，
1. 主要是线程池的使用是一方面，
2. 还有就是**线程安全**的一些问题。
3. 线程的创建记得只写核心参数不要无用参数。
好了，最重要的东西来了，为什么接口有时候正确有时候又不正确呢？

#### 思路：
前后代码看了看，只可能是多线程这里的使用出了问题：仔细看一下线程的代码有将queryParam这个
参数传入线程里重新set页然后数去查询。但是这里可是只有一个queryParam对象啊，那么多线程去setPage() 然后交给线程池，
你能线程保证自己去拿的时候是自己set的那个分页参数么？显然是有线程安全问题的。线程执行的时候不一定拿到的是自己set的分页参数！

初步定位到了问题我改善了下这小部分代码
验证的代码：
~~~java
public class Controller{
	
	@RequestMapping(value = "/exportExcel")
	@ResponseBody
	public String exportExcel(QueryParam queryParam, HttpServletResponse response) {
		param.setIsExport("export");
	    synchronized (this.exportLock){
            List<Map<String,Object>> resultList = new ArrayList<>();
            Long total = excelService.countOfficialMicroBlogPutList(queryParam);
            // 虽然500条能拉回来但是有的数据比较大，所以就取了100--------------------------------------
            params.setRows(100);
            int totalPage =  (int)(total/100) + 1;
            List<Future<List<Map<String, Object>>>> futures = new ArrayList<>();
            for (int i = 1; i <= totalPage; i++) {
                Future<List<Map<String, Object>>> submit = taskExecutor.submit(new ExcelWorker(i, queryParam));
                futures.add(submit);
            }
            for(Future<List<Map<String, Object>>> f : futures){
                try {
                    resultList.addAll(f.get());
                } catch (InterruptedException | ExecutionException e) {
                    logger.error("获取数据异常", e);
                }
            }
            //-----------------------------------简洁快速的代码-----------------------------------
	    }
		//........
	}
	
    private class ExcelWorker implements Callable{
		
		private int page;
        private QueryParam queryParam;
		
		ExcelWorker(int page, QueryParam queryParam) {
			this.page = page;
			this.queryParam = queryParam;
		}
		
		@Override
		public List<Map<String, Object>> call() {
			QueryParam params = new QueryParam();
			BeanUtils.copyProperties(pageParam, params);
			params.setPage(page);
			// 只取不变的部分
			try {
				PageBo<Map<String, Object>> threadPage = excelService.getOfficialMicroBlogPutList(params);
				return threadPage.getDataList();
			} catch (Exception e) {
				logger.error("分页查询异常", e);
			}
			return null;
		}
	}
}

~~~

先说结果：优化后的代码没有出现数据问题，响应也比较快,2s左右就可以看到表格导出了。完全符合要求了。
主要是以下几点改进：
1. 线程问题，安全这些线程都应该用自己的分页参数去请求，接口所以必须得clone(深度)或者copy一个新的对象才可以。
2. worker的优化：只要两个不可缺的关键参数(还有就是copy的任务交给线程，不要让主线程做这么无聊的事情)
3. 直接交个线程池处理后再去使用Future去取得结果。
4. 不用去shutdown线程池，如果你是自己创建的线程池，他自己会管理自己。

这篇文章对那些大佬是没什么提升的，对于没有接触过多线程，Callable,Future等等的人来说这些东西确实是比较棘手的,
但是只要认真分析出问题的代码行附近，认真分析对比还是比较快速就能得到答案的。









