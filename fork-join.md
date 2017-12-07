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
