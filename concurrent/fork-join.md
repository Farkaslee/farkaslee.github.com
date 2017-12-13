### Fork Join 介绍

如果让我们来设计一个Fork/Join框架，该如何设计？这个思考有助于你理解Fork/Join框架的设计。

第一步分割任务。首先我们需要有一个fork类来把大任务分割成子任务，有可能子任务还是很大，所以还需要不停的分割，直到分割出的子任务足够小。

第二步执行任务并合并结果。分割的子任务分别放在双端队列里，然后几个启动线程分别从双端队列里获取任务执行。子任务执行完的结果都统一放在一个队列里，启动一个线程从队列里拿数据，然后合并这些数据。

Fork/Join使用两个类来完成以上两件事情：

ForkJoinTask：我们要使用ForkJoin框架，必须首先创建一个ForkJoin任务。它提供在任务中执行fork()和join()操作的机制，通常情况下我们不需要直接继承ForkJoinTask类，而只需要继承它的子类，Fork/Join框架提供了以下两个子类：
RecursiveAction：用于没有返回结果的任务。
RecursiveTask ：用于有返回结果的任务。
ForkJoinPool ：ForkJoinTask需要通过ForkJoinPool来执行，任务分割出的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。当一个工作线程的队列里暂时没有任务时，它会随机从其他工作线程的队列的尾部获取一个任务。
### Fork Join 在项目中的实际使用
###实际使用代码
```java

@Component
public class OngoingPEER2PEERDataViewBuilder extends OngoingDataViewBuilder {
    @Autowired
    private OngoingAllInvestmentItemDataLoader allInvestmentItemDataLoader;
    @Autowired
    private TransferableDateBuilderV2 dateBuild;

    public final static ForkJoinPool peer2peerPool = new ForkJoinPool(200);

    private List<InvestmentItemBuilder> getAllApplicableBuilders(){
        List<InvestmentItemBuilder> builders = new ArrayList<InvestmentItemBuilder>();
        builders.add(getCtx().getBean(OngoingAYDInvestmentItemBuilder.class));
        builders.add(getCtx().getBean(OngoingP2PDJInvestmentItemBuilderV2.class));
        builders.add(getCtx().getBean("OngoingP2PInvestmentItemBatchBuilderV2",OngoingP2PInvestmentItemBatchBuilderV2.class));
        builders.add(getCtx().getBean(OngoingZJDInvestmentItemBuilderV2.class));
        builders.add(getCtx().getBean(OngoingDSJRInvestmentItemBuilderV2.class));
        builders.add(getCtx().getBean(OngoingNoGuarantorInvestmentItemBuilderV2.class));
        builders.add(getCtx().getBean(OngoingBXTInvestmentItemBuilder.class));
        builders.add(getCtx().getBean("OngoingLCLoanInvestmentItemBatchBuilderV2",OngoingLCLoanInvestmentItemBatchBuilderV2.class));
        builders.add(getCtx().getBean(OngoingJYPROInvestmentItemBuilder.class));
        return builders;
    }

    @Override
    protected List<OngoingInvestmentItem> getInvestmentItems(OngoingDataViewRequestContext requestContext) {
        List<OngoingInvestmentItem> ongoingInvestmentItems = new ArrayList<OngoingInvestmentItem>();

        List<String> productCategories = Lists.newArrayList();
        List<InvestmentItemBuilder> investmentItemBuilders = getAllApplicableBuilders();
        for(InvestmentItemBuilder investmentItemBuilder : investmentItemBuilders){
            productCategories.addAll(investmentItemBuilder.getApplicableProductCategory());
        }

        ListMultimap<InvestmentItemBuilder, InvestmentDTO> builderInvestmentItems = ArrayListMultimap.create();

        List<Investment> investments = allInvestmentItemDataLoader.loadData(requestContext.getUserId(),
                productCategories, requestContext.getAssetFilterType());
        if(CollectionUtils.isNotEmpty(investments)){
            for (Investment investment : investments){
                if(StringUtils.equals(investment.IN_AGREEMENT,"1"))
                    continue;
                InvestmentDTO investmentDTO = new InvestmentDTO(investment);
                for(InvestmentItemBuilder investmentItemBuilder : investmentItemBuilders)
                    if(investmentItemBuilder.isValid(investmentDTO))
                        builderInvestmentItems.put(investmentItemBuilder, investmentDTO);
            }
        }

        Set<OngoingItemBuilderForkJoinTask> forkJoinTasks = new HashSet<OngoingItemBuilderForkJoinTask>();
        for(InvestmentItemBuilder investmentItemBuilder : investmentItemBuilders){
            OngoingItemBuilderForkJoinTask forkJoinTask = new OngoingItemBuilderForkJoinTask(investmentItemBuilder, builderInvestmentItems);
            forkJoinTasks.add(forkJoinTask);
        }

        try{
            List<Future<List<OngoingInvestmentItem>>> investmentItemFutures = peer2peerPool.invokeAll(forkJoinTasks);
            for(Future<List<OngoingInvestmentItem>> investmentItemFuture: investmentItemFutures){
                ongoingInvestmentItems.addAll(investmentItemFuture.get());
            }
        } catch (Exception e){
            Logger.error(this,String.format("Peer2peerPool exception: %s",e.getMessage()));
            return ongoingInvestmentItems;
        }

        return ongoingInvestmentItems;
    }
   
}
```
OngoingItemBuilderForkJoinTask class

```java
public class OngoingItemBuilderForkJoinTask implements Callable<List<OngoingInvestmentItem>> {

    private InvestmentItemBuilder builder;
    private ListMultimap<InvestmentItemBuilder, InvestmentDTO> builderInvestmentMap = ArrayListMultimap.create();
    OngoingItemBuilderForkJoinTask(InvestmentItemBuilder builder,ListMultimap<InvestmentItemBuilder, InvestmentDTO> builderInvestmentMap){
        this.builder = builder;
        this.builderInvestmentMap = builderInvestmentMap;
    }
    @Override
    public List<OngoingInvestmentItem> call() throws Exception {
        return builder.build(builderInvestmentMap.get(builder));
    }
}
```


可以参考这里的原理说明：
http://www.infoq.com/cn/articles/fork-join-introduction
