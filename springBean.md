
容器启动时：

最先调用

        BeanFactoryPostProcessor

                        ->postProcessBeanFactory() 

 

getBean时：

实例化之后调用：

        InstantiationAwareBeanPostProcessor

                ->postProcessPropertyValues()

 

初始化时：

        属性注入（setter）

 

        BeanNameAware

                ->setBeanName() 

        BeanFactoryAware

                ->setBeanFactory() 

        ApplicationContextAware

                ->setApplicationContext()

  

        BeanPostProcessor

               ->postProcessBeforeInitialization()

        InitializingBean

                ->afterPropertiesSet()

        init-method属性

        BeanPostProcessor

                ->postProcessAfterInitialization()

 

        DiposibleBean

                ->destory() 

        destroy-method属性
