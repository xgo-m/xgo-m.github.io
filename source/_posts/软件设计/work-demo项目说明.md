---
title: work-demo项目说明
date: 2023-06-16 17:41:00
author: xgo-m
img: https://statics.sh1a.qingstor.com/2018/11/24/design.jpg
top: true
categories: 软件设计
tags:
  - 面向对象编程
  - Java
---

<h2 align="center">work-demo项目说明</h2>

# 项目框架介绍
work-demo项目主要有两个端：（后台管理端）`work-demo-admin`、（小程序端）`work-demo-mp`，还有一个核心的通用包`work-demo-common`
## 后端主要使用的技术
* Spring Boot 2.7.4：使用springboot用来简化新Spring应用的初始搭建以及开发过程
* Spring Security 5.7.3 ：Spring Security是一个专注于为 Java 应用程序提供身份验证和授权的框架
* Spring Data JPA 2.7.4：Spring Data JPA是基于 JPA 规范的上层封装，旨在简化 JPA 的使用。
* QueryDSL 5.0.0：QueryDSL是一种流行的 Java 查询框架，它以编程方式构建类型安全的查询。
* jackson-databind：数据绑定包， 基于"对象绑定" 解析的 API 和"树模型"解析的 API 依赖基于"流模式"解析的 API。

## 核心部分介绍

### 自定义序列化器
部分核心代码
该类是一个ObjectMapper的构建器，用于构建一个可自定义配置的ObjectMapper对象。主要实现了以下功能：

1. 通过Jackson2ObjectMapperBuilder对象进行配置，包括禁用XML映射器、使用Java 8时间模块来处理日期和时间类型、不报告未知属性错误、使用自定义JsonInclude的配置方式来解决Hibernate代理对象的序列化问题等。
2. 通过serializerByType方法和JpaLazyInitializeObjectSerializer类的实现，实现了JPA关联对象集合的序列化控制，避免在序列化时触发Hibernate的懒加载机制。
3. 通过postConfigurer方法和CustomBeanSerializerModifier类的实现，实现了对序列化器的后处理，可以自定义修改序列化器的行为。

该类主要方法包括serialize方法，用于对JPA关联对象集合进行序列化控制，避免在序列化时触发Hibernate的懒加载机制。该类可以作为一个工具类，被其他类调用，以构建一个可自定义配置的ObjectMapper对象，用于序列化和反序列化Java对象和JSON字符串之间的转换。
```java
public class ObjectMapperBean {
    public static final Jackson2ObjectMapperBuilder BUILDER;

    static {
        BUILDER =
                new Jackson2ObjectMapperBuilder()
                        .createXmlMapper(false) //禁用XML映射器
                        .modules(new JavaTimeModule()) //使用Java 8时间模块来处理日期和时间类型
                        .failOnUnknownProperties(false) //不报告未知属性错误
                        .serializationInclusion( //使用JsonInclude来配置序列化的包含规则
                                JsonInclude.Value.construct(JsonInclude.Include.CUSTOM, JsonInclude.Include.CUSTOM)
                                        //使用自定义JsonInclude的配置方式来解决Hibernate代理对象的序列化问题
                                        .withValueFilter(JsonIncludeValueFilter.class))
                        .serializerByType(//通过类型来注册自定义的序列化器
                                PersistentCollection.class, JpaLazyInitializeObjectSerializer.INSTANCE)
                        .serializerByType(HibernateProxy.class, JpaLazyInitializeObjectSerializer.INSTANCE)
                        .featuresToEnable(DeserializationFeature.READ_UNKNOWN_ENUM_VALUES_AS_NULL)
                        .timeZone(TimeZone.getDefault())
                        .deserializerByType(
                                LocalDateTime.class,
                                new LocalDateTimeDeserializer(DateTimeFormatter.ISO_LOCAL_DATE_TIME))
                        .serializerByType(
                                LocalDateTime.class,
                                new LocalDateTimeSerializer(DateTimeFormatter.ISO_LOCAL_DATE_TIME))
                        .postConfigurer(
                                objectMapper ->
                                        objectMapper.setSerializerFactory(
                                                objectMapper
                                                        .getSerializerFactory()
                                                        .withSerializerModifier(new CustomBeanSerializerModifier())));

        INSTANCE = BUILDER.build();
    }


    /** 使jpa的关联对象集合不序列化 */
    public static class JpaLazyInitializeObjectSerializer extends JsonSerializer<Object> {
        
        @Override
        public void serialize(Object value, JsonGenerator gen, SerializerProvider serializers)
                throws IOException {
            if (value instanceof PersistentCollection) {//首先判断要序列化的对象是否是PersistentCollection类型的对象
                if (((PersistentCollection) value).wasInitialized()) {//检查该对象是否已经被初始化
                    if (value instanceof Collection) {//判断集合
                        serializers.findValueSerializer(Collection.class).serialize(value, gen, serializers);
                    } else if (value instanceof Map) {
                        serializers.findValueSerializer(Map.class).serialize(value, gen, serializers);
                    }
                } else {
                    serializers.defaultSerializeNull(gen);//如果没有被初始化，则序列化为null，避免在序列化时触发Hibernate的懒加载机制。
                }
            } else if (value instanceof HibernateProxy) {//如果要序列化的对象是HibernateProxy类型的对象
                if (((HibernateProxy) value).getHibernateLazyInitializer().isUninitialized()) {//检查它是否已经被初始化
                    serializers.defaultSerializeNull(gen);//如果没有被初始化，则序列化为null，避免在序列化时触发Hibernate的懒加载机制。
                } else {
                    var writeReplace = ((HibernateProxy) value).writeReplace();//如果已经被初始化，则使用代理对象的writeReplace()方法获取实际的值，并使用findValueSerializer()方法获取该值的序列化器进行序列化操作
                    serializers
                            .findValueSerializer(writeReplace.getClass())
                            .serialize(writeReplace, gen, serializers);
                }
            } else {
                serializers.defaultSerializeValue(value, gen);//如果要序列化的对象既不是PersistentCollection类型，也不是HibernateProxy类型，则按照默认方式进行序列化，使用defaultSerializeValue()方法将对象序列化为JSON格式。
            }
        }
    }

    public static class CustomBeanSerializerModifier extends BeanSerializerModifier {

        /**当序列化对象时，带有这些注释的属性就会使用自定义的序列化器来处理这些属性的序列化操作，解决了Hibernate代理对象和关联对象集合的序列化问题。最后，该方法返回更新后的属性列表。**/
        @Override
        public List<BeanPropertyWriter> changeProperties(
                SerializationConfig config,
                BeanDescription beanDesc,
                List<BeanPropertyWriter> beanProperties) {
            for (BeanPropertyWriter beanProperty : beanProperties) {//该方法首先遍历所有的属性，判断是否带有ManyToMany、OneToMany、ManyToOne或OneToOne注释之一
                if (beanProperty.getAnnotation(ManyToMany.class) != null
                        || beanProperty.getAnnotation(OneToMany.class) != null
                        || beanProperty.getAnnotation(ManyToOne.class) != null
                        || beanProperty.getAnnotation(OneToOne.class) != null) {
                    beanProperty.assignSerializer(JpaLazyInitializeObjectSerializer.INSTANCE);//如果是，则为该属性分配JpaLazyInitializeObjectSerializer.INSTANCE实例作为其自定义序列化器
                }
            }
            return beanProperties;
        }
    }
}
```
这个类提供了一个预配置的 Jackson 库中的 ObjectMapper 类的实例，它的目的是用于序列化和反序列化 JSON 数据，并解决了 Hibernate 代理对象和懒加载 JPA 实体集合的序列化问题。具体而言，它通过自定义序列化器和过滤器来处理这些问题。

创建Jackson 对象映射器配置进行配置上述自定义序列化器
```java
@Configuration
public class JacksonObjectMapperConfiguration {
  @Bean
  public Jackson2ObjectMapperBuilder jackson2ObjectMapperBuilder() {
    return ObjectMapperBean.BUILDER;
  }

  @Bean
  public ObjectMapper objectMapper() {
    return ObjectMapperBean.INSTANCE;
  }
}
```

使用 通过引入自定义的基础配置进行使用
```java
/** 引入自定义的基础配置 */
@Import({
  HibernatePropertiesConfig.class,
  JacksonObjectMapperConfiguration.class,
  GlobalExceptionHandler.class
})
@Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
@Target({java.lang.annotation.ElementType.TYPE})
@EntityScan(basePackageClasses = BaseEntity.class)
@EnableJpaRepositories(
    basePackageClasses = BaseJpaRepository.class,
    repositoryFactoryBeanClass = CustomJpaRepositoryFactoryBean.class)
public @interface EnableBaseConfiguration {}
```

### JPA部分封装

#### 自定义投影接口
创建自定义投影接口
```java
public interface DynamicProjectionRepository<T, ID> {
  <R> List<R> findAllProjection(Predicate predicate, ConstructorExpression<R> projectionPath);

  <R> List<R> findAllProjection(Predicate predicate, Expression<R> expression);

  <R> Optional<R> findOneProjection(Predicate predicate, ConstructorExpression<R> expression);

  <R> Optional<R> findOneProjection(Predicate predicate, Expression<R> projectionPath);

  Optional<Tuple> findOne(Predicate predicate, Expression<?>... expressions);

  Optional<T> findById(ID id, String... attributes);

  String getEntityName();
}
```
实现自定义的投影接口并继承QuerydslJpaPredicateExecutor类，这个类提供了一个基本的查询方法，可以根据给定的条件查询实体并返回它们的列表。
```java
public class DynamicProjectionRepositoryImpl<T, ID> extends QuerydslJpaPredicateExecutor<T>
    implements DynamicProjectionRepository<T, ID> {

  private final EntityPath<T> path;
  private final EntityManager entityManager;
  private final JpaEntityInformation<T,?> entityInformation;

  public DynamicProjectionRepositoryImpl(
      JpaEntityInformation<T, ?> entityInformation,
      EntityManager entityManager,
      EntityPathResolver resolver,
      CrudMethodMetadata metadata) {
    super(entityInformation, entityManager, resolver, metadata);
    this.path = resolver.createPath(entityInformation.getJavaType());
    this.entityManager = entityManager;
    this.entityInformation = entityInformation;
  }

  @Override
  public <R> List<R> findAllProjection(
      Predicate predicate, ConstructorExpression<R> projectionPath) {
    return createQuery(predicate).select(projectionPath).fetch();
  }

  @Override
  public <R> Optional<R> findOneProjection(Predicate predicate, Expression<R> projectionPath) {
    return Optional.ofNullable(createQuery(predicate).select(projectionPath).fetchOne());
  }

  @Override
  public Optional<Tuple> findOne(Predicate predicate, Expression<?>... expressions) {
    return Optional.ofNullable(createQuery(predicate).select(expressions).fetchOne());
  }



  @Override
  public <R> Optional<R> findOneProjection(
      Predicate predicate, ConstructorExpression<R> expression) {
    return Optional.ofNullable(createQuery(predicate).select(expression).fetchOne());
  }

  @Override
  public Optional<T> findById(ID id, String... attributes) {
    EntityGraph<? extends T> entityGraph = entityManager.createEntityGraph(path.getType());
    entityGraph.addAttributeNodes(attributes);
    return Optional.ofNullable(
        entityManager.find(path.getType(), id, Map.of(QueryHints.HINT_FETCHGRAPH, entityGraph)));
  }


  @Override
  public <R> List<R> findAllProjection(Predicate predicate, Expression<R> expression) {
    return createQuery(predicate).select(expression).fetch();
  }

  @Override
  public String getEntityName() {
    return entityInformation.getEntityName();
  }
}
```


该类是一个自定义的JpaRepositoryFactory，继承自JpaRepositoryFactory，用于扩展JPA的默认行为。主要实现了以下功能：

1. 重写了JpaRepositoryFactory的getRepositoryFragments方法，用于返回一个扩展了DynamicProjectionRepository的RepositoryFragments。
2. 判断当前的RepositoryMetadata是否继承了DynamicProjectionRepository接口，如果是，则将DynamicProjectionRepositoryImpl的代码片段添加到RepositoryFragments中，以扩展该Repository的默认行为。

该类主要方法是getRepositoryFragments方法，用于返回一个扩展了DynamicProjectionRepository的RepositoryFragments。它可以作为一个工厂类，被其他类调用，以创建一个扩展了DynamicProjectionRepository的Repository。该类可以用于扩展JPA的默认行为，实现更灵活的数据操作方式。
```java
/**
 * 自定义的Jpa存储工厂（主要将JpaRepositoryFactory的getRepositoryFragments重写，主要参考原方法）
 */
public class CustomJpaRepositoryFactory extends JpaRepositoryFactory {
  
  public CustomJpaRepositoryFactory(EntityManager entityManager) {
    super(entityManager);
  }
  
  @Override
  protected RepositoryComposition.@NotNull RepositoryFragments getRepositoryFragments(
          @NotNull RepositoryMetadata metadata,
          @NotNull EntityManager entityManager,
          @NotNull EntityPathResolver resolver,
          @NotNull CrudMethodMetadata crudMethodMetadata) {
    var repositoryFragments =
        super.getRepositoryFragments(metadata, entityManager, resolver, crudMethodMetadata);
    //用来判断当前Class 对象所表示的类或接口与指定的 Class 参数所表示的类或接口是否相同
    //此处参考了JpaRepositoryFactory的getRepositoryFragments实现将原QuerydslPredicateExecutor换成了我们自定义的DynamicProjectionRepository类
    if (DynamicProjectionRepository.class.isAssignableFrom(metadata.getRepositoryInterface())) {
      repositoryFragments =
          repositoryFragments.append(
              RepositoryComposition.RepositoryFragments.just(
                      //此处也是将我们自定义的DynamicProjectionRepositoryImpl的代码片段添加到repositoryFragments 中
                  new DynamicProjectionRepositoryImpl<>(
                      getEntityInformation(metadata.getDomainType()),
                      entityManager,
                      resolver,
                      crudMethodMetadata)));
    }
    return repositoryFragments;
  }
}

```
创建CustomJpaRepositoryFactoryBean来使用上面创建的自定义的Jpa存储工厂
```java
/** CustomJpaRepositoryFactoryBean类重写了JpaRepositoryFactoryBean的一些主要方法并进行部分自定义 **/
public class CustomJpaRepositoryFactoryBean<T extends Repository<S, ID>, S, ID>
    extends JpaRepositoryFactoryBean<T, S, ID> {

    /**
     *使用自己定义的CustomJpaRepositoryFactory
     */
    @Override
    protected RepositoryFactorySupport createRepositoryFactory(EntityManager entityManager) {
        JpaRepositoryFactory jpaRepositoryFactory = new CustomJpaRepositoryFactory(entityManager);
        jpaRepositoryFactory.setEntityPathResolver(entityPathResolver);
        jpaRepositoryFactory.setEscapeCharacter(escapeCharacter);

        if (queryMethodFactory != null) {
            jpaRepositoryFactory.setQueryMethodFactory(queryMethodFactory);
        }

        return jpaRepositoryFactory;
    }
}
```
使用 通过引入自定义的基础配置进行使用
```java
/** 引入自定义的基础配置 */
@Import({
  HibernatePropertiesConfig.class,
  JacksonObjectMapperConfiguration.class,
  GlobalExceptionHandler.class
})
@Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
@Target({java.lang.annotation.ElementType.TYPE})
@EntityScan(basePackageClasses = BaseEntity.class)
@EnableJpaRepositories(
    basePackageClasses = BaseJpaRepository.class,
    repositoryFactoryBeanClass = CustomJpaRepositoryFactoryBean.class)
public @interface EnableBaseConfiguration {}
```
#### 自定义 Hibernate 集成器
创建自定义 Hibernate 集成器提供程序
这个类提供了一个自定义的 IntegratorProvider 实现，用于为 Hibernate 添加一些自定义的集成器，主要的作用是处理运行时生成的 Hibernate 代理对象和懒加载的 JPA 实体集合的更新问题。
```java
public class CustomHibernateIntegratorProvider implements IntegratorProvider {
  @Override
  public List<Integrator> getIntegrators() {
    return List.of(
        new Integrator() {
          @Override
          public void integrate(
              Metadata metadata,
              SessionFactoryImplementor sessionFactory,
              SessionFactoryServiceRegistry serviceRegistry) {
            // 自定义MergeEventListener
            var eventListenerRegistry = serviceRegistry.getService(EventListenerRegistry.class);
            eventListenerRegistry.setListeners(
                EventType.MERGE, new CustomDefaultMergeEventListener());
            // 用于处理运行时生成HibernateProxy
            var service = new BytecodeProviderImpl(ClassFileVersion.ofThisVm());
            serviceRegistry
                .locateServiceBinding(BytecodeProvider.class)
                .setService(service);
            serviceRegistry
                .locateServiceBinding(ProxyFactoryFactory.class)
                .setService(service.getProxyFactoryFactory());
          }

          @Override
          public void disintegrate(
              SessionFactoryImplementor sessionFactory,
              SessionFactoryServiceRegistry serviceRegistry) {}
        });
  }

  /** CustomDefaultMergeEventListener 类是 DefaultMergeEventListener 的一个子类，它重写了 copyValues() 方法，用于处理 Hibernate 中的更新操作。
   * 在 copyValues() 方法中，它首先获取了实体的原始值和目标值，然后根据属性的 @Column 注解中的 nullable 配置，设置实体属性的值，如果属性在原始值中为 null，
   * 而且不允许为 null，则使用目标值代替。这样做的目的是为了避免在更新实体时，将 null 值赋给不允许为 null 的属性，从而导致错误。 **/
  public static class CustomDefaultMergeEventListener extends DefaultMergeEventListener {
    @Override
    protected void copyValues(
        EntityPersister persister,
        Object entity,
        Object targetEntity,
        SessionImplementor session,
        Map copyCache) {
      var original = persister.getPropertyValues(entity);
      var target = persister.getPropertyValues(targetEntity);
      var types = persister.getPropertyTypes();
      // 获取列是否可以为空, 即@Column中的nullable配置
      var propertyNullability = persister.getPropertyNullability();

      Object[] copied = new Object[original.length];
      for (int i = 0; i < types.length; i++) {
        // 修改此处的判断条件, 你可以不验证@Column注解的nullable
        // 下面这种用法既可以给可以为null的字段设置null, 又不会更新不能为null的字段
        if (!propertyNullability[i] && original[i] == null) {
          // 如果nullable为false, 并且传入的值为null, 则使用原数据, 即不更新null
          copied[i] = target[i];
        } else
        // 下面这部分是原有的代码, 不做改动
        if (original[i] == LazyPropertyInitializer.UNFETCHED_PROPERTY
            || original[i] == PropertyAccessStrategyBackRefImpl.UNKNOWN) {
          copied[i] = target[i];
        } else if (target[i] == LazyPropertyInitializer.UNFETCHED_PROPERTY) {
          copied[i] = types[i].replace(original[i], null, session, targetEntity, copyCache);
        } else {
          copied[i] = types[i].replace(original[i], target[i], session, targetEntity, copyCache);
        }
      }
      persister.setPropertyValues(targetEntity, copied);
    }
  }
}

```
HibernatePropertiesCustomizer 接口，并重写了 customize() 方法。它的主要目的是为 Hibernate 配置自定义的集成器，以便在 Hibernate 启动时自动加载它们。
```java
@Component
public class HibernatePropertiesConfig implements HibernatePropertiesCustomizer {
  @Override
  public void customize(Map<String, Object> hibernateProperties) {
    hibernateProperties.put(
        "hibernate.integrator_provider", new CustomHibernateIntegratorProvider());
  }
}
```

使用 通过引入自定义的基础配置进行使用
```java
/** 引入自定义的基础配置 */
@Import({
  HibernatePropertiesConfig.class,
  JacksonObjectMapperConfiguration.class,
  GlobalExceptionHandler.class
})
@Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
@Target({java.lang.annotation.ElementType.TYPE})
@EntityScan(basePackageClasses = BaseEntity.class)
@EnableJpaRepositories(
    basePackageClasses = BaseJpaRepository.class,
    repositoryFactoryBeanClass = CustomJpaRepositoryFactoryBean.class)
public @interface EnableBaseConfiguration {}
```

### 封装常用的CRUD

#### BaseJpaRepository
该接口主要继承了下列接口
* JpaRepository:JpaRepository是Spring Data JPA提供的接口之一，提供了许多基本的数据访问方法，如保存、删除、查找等。
* JpaSpecificationExecutor:JpaSpecificationExecutor也是Spring Data JPA提供的接口之一，它提供了一种在运行时动态构建查询条件的方法，这些查询条件可以是任意的复杂条件，如AND、OR等。
* QuerydslBinderCustomizer:QuerydslBinderCustomizer是Querydsl提供的接口之一，它允许我们在自定义查询中使用自定义绑定。通过实现该接口，我们可以在运行时绑定自定义类型，以支持Querydsl查询中的类型安全。具体地说，它使用Querydsl Binder API来实现自定义绑定，Binder API提供了一种类型安全的方式来将Java对象映射到Querydsl路径表达式。
* QuerydslPredicateExecutor:QuerydslPredicateExecutor也是Querydsl提供的接口之一，它提供了一种在运行时动态构建查询条件的方法，这些查询条件可以是任意的复杂条件，如AND、OR等。与JpaSpecificationExecutor不同的是，它使用Querydsl的Predicate接口来表示查询条件。
* DynamicProjectionRepository:这里继承 DynamicProjectionRepository有两个优势一是可以通过继承该接口获得其接口的功能，二继承该接口后可以在CustomJpaRepositoryFactory.getRepositoryFragments中的`DynamicProjectionRepository.class.isAssignableFrom(metadata.getRepositoryInterface())`返回true进行加载DynamicProjectionRepository的实现的代码片段
```java
@NoRepositoryBean
public interface BaseJpaRepository<T extends BaseEntity, E extends EntityPath<T>>
    extends JpaRepository<T, Long>, //JpaRepository是Spring Data JPA提供的接口之一，提供了许多基本的数据访问方法，如保存、删除、查找等。
        JpaSpecificationExecutor<T>,//JpaSpecificationExecutor也是Spring Data JPA提供的接口之一，它提供了一种在运行时动态构建查询条件的方法，这些查询条件可以是任意的复杂条件，如AND、OR等。
        QuerydslBinderCustomizer<E>,//QuerydslBinderCustomizer是Querydsl提供的接口之一，它允许我们在自定义查询中使用自定义绑定。通过实现该接口，我们可以在运行时绑定自定义类型，以支持Querydsl查询中的类型安全。具体地说，它使用Querydsl Binder API来实现自定义绑定，Binder API提供了一种类型安全的方式来将Java对象映射到Querydsl路径表达式。
        QuerydslPredicateExecutor<T>,//QuerydslPredicateExecutor也是Querydsl提供的接口之一，它提供了一种在运行时动态构建查询条件的方法，这些查询条件可以是任意的复杂条件，如AND、OR等。与JpaSpecificationExecutor不同的是，它使用Querydsl的Predicate接口来表示查询条件。
        DynamicProjectionRepository<T, Long> //这里继承 DynamicProjectionRepository有两个优势一是可以通过继承该接口获得其接口的功能，二继承该接口后可以在CustomJpaRepositoryFactory.getRepositoryFragments中的【DynamicProjectionRepository.class.isAssignableFrom(metadata.getRepositoryInterface())】返回true进行加载DynamicProjectionRepository的实现的代码片段
{

/**
 * 这段代码的作用是为LocalDateTime类型定义一个过滤器，使得在使用Querydsl进行查询时，可以方便地对LocalDateTime类型进行过滤。
 */
  @Override
  default void customize(QuerydslBindings bindings, @NotNull E root) {
      //将LocalDateTime类型绑定到Querydsl DateTimePath类型上
    bindings
        .bind(LocalDateTime.class)
        .all(
            (DateTimePath<LocalDateTime> path, Collection<? extends LocalDateTime> value) -> {
                //首先检查集合是否为空，如果是，则返回一个空的Optional对象
              if (value.isEmpty()) {
                return Optional.empty();
              }
              Iterator<? extends LocalDateTime> iterator = value.iterator();
              //如果集合中只有一个元素，则返回一个Optional对象
              if (value.size() == 1) {
                return Optional.of(path.after(iterator.next()));
              }
              return Optional.of(path.between(iterator.next(), iterator.next()));
            });
  }

  default boolean exists(T entity) {
    return exists(Example.of(entity));
  }

  default Optional<T> findOne(T example) {
    return findOne(Example.of(example));
  }

  @Override
  @NotNull
  List<T> findAll(Predicate predicate);

  @Override
  @NotNull
  List<T> findAll(Predicate predicate, Sort sort);

  @Override
  @NotNull
  List<T> findAll(Predicate predicate, OrderSpecifier<?>... orders);
}

```

#### BaseService
该类是一个基础服务类，提供了一些通用的操作数据库的方法和一些验证逻辑，主要实现了以下功能：

1. 使用泛型T表示实体类的类型，使用泛型R表示实体类对应的JpaRepository类型，这样可以让该类支持操作不同的实体类和对应的JpaRepository。
2. 提供了一些基本的数据库操作方法，如findAll、findById、findByIds、findAll、findOne、findAll、create、update、deleteById、delete、deleteAll，用于查询、创建、更新和删除实体对象，这些操作都是通过调用repository的方法实现的。
3. 提供了一个patchUpdate方法，用于部分更新实体对象的属性值，该方法会首先从数据库中查找到对应的实体对象，然后根据传入的Map中的属性值更新实体对象，最后调用update方法将更新后的实体对象保存回数据库中。
4. 提供了一些验证逻辑的方法，如assertBizException、assertUserException和validateEntity，用于验证业务逻辑和用户请求参数的合法性，如果验证失败，则会抛出异常。
5. 提供了一些辅助方法，如clearRelations，用于清除实体对象中的关联关系，以及entityManager的设置和validator的初始化。

该类主要方法包括findAll、findById、create、update、deleteById等基本的数据库操作方法，以及patchUpdate和validateEntity等验证逻辑的方法，它可以作为其他服务类的基类来复用这些通用的操作方法和验证逻辑，减少代码重复，提高开发效率。

```java
public class BaseService<
    T extends BaseEntity, R extends BaseJpaRepository<T, ? extends EntityPath<T>>> {
  protected R repository;

  protected EntityManager entityManager;

  @SuppressWarnings("resource")
  protected Validator validator =
      Validation.byDefaultProvider()
          .configure()
          .addProperty("hibernate.validator.fast_fail", "true")//快速模式验证过程中只要有一个失败，则返回验证失败信息。
          .buildValidatorFactory()
          .getValidator();

  @Autowired
  public void setEntityManager(EntityManager entityManager) {
    this.entityManager = entityManager;
  }

  @SuppressWarnings({"SpringJavaInjectionPointsAutowiringInspection"})
  @Autowired
  public void setRepository(R repository) {
    this.repository = repository;
  }

  public List<T> findAll() {
    return repository.findAll();
  }

  public T findById(Long id) {
    return repository
        .findById(id)
        .orElseThrow(
            () ->
                new BizException(
                    BizStatus.REQUESTED_RESOURCE_DOES_NOT_EXIST,
                    "No %s value of %d present".formatted(repository.getEntityName(), id)));
  }

  public List<T> findByIds(Iterable<Long> ids) {
    return repository.findAllById(ids);
  }

  public List<T> findAll(Predicate predicate) {
    return repository.findAll(predicate);
  }

  public T findOne(Predicate predicate) {
    return repository.findOne(predicate).orElseThrow();
  }

  public Page<T> findAll(Predicate predicate, Pageable pageable) {
    return repository.findAll(predicate, pageable);
  }

  public T create(T entity) {
    if (entity.getId() != null) {
      throw new UserRequestParameterException("new entity id must be null");
    }
    return repository.save(entity);
  }

  public T update(Long id, T entity) {
    entity.setId(id);
    return repository.save(entity);
  }

  public T patchUpdate(Long id, Map<String, Object> patch) {
    var obj = repository.findById(id).orElseThrow();
    patch.forEach(
        (k, v) -> {
          var prop = BeanUtils.getPropertyDescriptor(obj.getClass(), k);
          if (prop == null) {
            return;
          }
          if (prop.getWriteMethod() == null) {
            return;
          }
          try {
            prop.getWriteMethod().invoke(obj, v);
          } catch (IllegalAccessException | InvocationTargetException e) {
            throw new BizException(BizStatus.SERVER_INTERNAL_ERROR, e.getMessage());
          }
        });
    return update(id, obj);
  }

  public void deleteById(Long id) {
    repository.deleteById(id);
  }

  public void delete(T entity) {
    repository.delete(entity);
  }

  public void deleteAll(Iterable<T> entities) {
    repository.deleteAll(entities);
  }

  public void deleteAllById(Iterable<Long> ids) {
    repository.deleteAllById(ids);
  }

  protected final void assertBizException(boolean condition, BizStatus bizStatus, String message) {
    if (!condition) {
      throw new BizException(bizStatus, message);
    }
  }

  protected final void assertUserException(boolean condition, String message) {
    if (!condition) {
      throw new UserRequestParameterException(message);
    }
  }

  protected final void assertBizException(boolean condition, BizStatus bizStatus) {
    if (!condition) {
      throw new BizException(bizStatus);
    }
  }

  protected final <A extends BaseEntity> A clearRelations(A entity) {
    entityManager.detach(entity);
    entity.clearRelations();
    return entity;
  }

  protected final <A extends BaseEntity> List<A> clearRelations(List<A> entities) {
    entities.forEach(this::clearRelations);
    return entities;
  }

  protected final <A extends BaseEntity> void validateEntity(A entity, Class<?>... groups) {
    var validate = validator.validate(entity, groups);
    if (validate.isEmpty()) {
      return;
    }
    var next = validate.iterator().next();
    throw new UserRequestParameterException(
        "%s: %s".formatted(next.getPropertyPath(), next.getMessage()));
  }
}
```
#### BaseController
该类是一个基础控制器类，用于处理HTTP请求，并将请求转发给对应的服务类进行处理。主要实现了以下功能：

1. 使用泛型T表示实体类的类型，使用泛型S表示实体类对应的服务类的类型，这样可以让该类支持操作不同的实体类和对应的服务类。
2. 提供了get、post、put、patch和delete等方法，用于处理HTTP请求，并将请求转发给对应的服务类进行处理。这些方法的实现都是通过调用服务类的对应方法实现的。
3. 提供了supportUnpaged方法，用于判断该控制器是否支持分页查询，如果不支持，则在不带分页参数的查询中自动分页。
4. 提供了ok和ok(body)方法，用于返回HTTP响应，其中ok方法返回一个空的响应，ok(body)方法返回一个包含指定body的响应。

该类主要方法包括get、post、put、patch和delete等处理HTTP请求的方法，它可以作为其他控制器类的基类来复用这些通用的HTTP请求处理方法，减少代码重复，提高开发效率。

```java
public class BaseController<
        T extends BaseEntity,
        S extends BaseService<T, ? extends BaseJpaRepository<T, ? extends EntityPath<T>>>> {
    protected S service;

    @SuppressWarnings({"SpringJavaInjectionPointsAutowiringInspection"})
    @Autowired
    public void setService(S service) {
        this.service = service;
    }

    public ResponseEntity<?> get(Predicate predicate, Pageable pageable) {
        if (pageable.equals(Pageable.unpaged())) {
            if (!supportUnpaged()) {
                pageable = PageRequest.of(0, 10);
            }
        }
        return ResponseEntity.ok(service.findAll(predicate, pageable));
    }

    public ResponseEntity<?> get(Long id) {
        return ResponseEntity.ok(service.findById(id));
    }

    public ResponseEntity<?> post(T entity) {
        return ResponseEntity.ok(service.create(entity));
    }

    public ResponseEntity<?> patch(Long id, Map<String, Object> patch) {
        return ResponseEntity.ok(service.patchUpdate(id, patch));
    }

    public ResponseEntity<?> put(Long id, T entity) {
        return ResponseEntity.ok(service.update(id, entity));
    }

    public ResponseEntity<?> delete(Long id) {
        service.deleteById(id);
        return ResponseEntity.ok().build();
    }

    protected boolean supportUnpaged() {
        return false;
    }

    protected final ResponseEntity.BodyBuilder ok() {
        return ResponseEntity.ok();
    }

    protected final <D> ResponseEntity<D> ok(D body) {
        return ResponseEntity.ok(body);
    }
}

```

#### RequestMappingDefinition
该接口是一个请求映射定义接口，提供了HTTP请求映射的方法定义，主要实现了以下功能：

1. 使用泛型T表示实体类的类型，该泛型可以在实现该接口的类中指定具体的实体类类型。
2. 提供了get、post、put、patch和delete等方法定义，用于定义HTTP请求映射的方法和参数，这些方法和参数的实现都是通过实现该接口的类进行实现的。
3. 为了保证请求映射的规范性，这些方法中的参数都采用了注解的方式进行定义，如@GetMapping("{id:\\d+}")中的@PathParam注解用于指定id参数的取值范围为数字。

该接口主要方法包括get、post、put、patch和delete等方法的定义，它可以作为控制器类的一个通用接口来复用这些HTTP请求映射的方法定义，减少代码重复，提高开发效率。

```java
public interface RequestMappingDefinition<T> {

  @GetMapping
  default ResponseEntity<?> get(Predicate predicate, Pageable pageable) {
    throw new UnsupportedOperationException();
  }

  @GetMapping("{id:\\d+}")
  default ResponseEntity<?> get(@PathVariable Long id) {
    throw new UnsupportedOperationException();
  }

  @PostMapping
  default ResponseEntity<?> post(
      @RequestBody @Validated(BaseEntity.CreateValidateGroup.class) T entity) {
    throw new UnsupportedOperationException();
  }

  @PutMapping("{id:\\d+}")
  default ResponseEntity<?> put(
      @PathVariable("id") Long id,
      @Validated(BaseEntity.UpdateValidateGroup.class) @RequestBody T entity) {
    throw new UnsupportedOperationException();
  }

  @PatchMapping("{id:\\d+}")
  default ResponseEntity<?> patch(
      @PathVariable("id") Long id, @RequestBody Map<String, Object> patch) {
    throw new UnsupportedOperationException();
  }

  @DeleteMapping("{id:\\d+}")
  default ResponseEntity<?> delete(@PathVariable("id") Long id) {
    throw new UnsupportedOperationException();
  }
}
```
### 身份验证和授权 Security

#### Spring Security 的配置类
这是一个 Spring Security 的配置类，用于配置应用程序的安全性。该类使用了 Spring 的注解来声明依赖关系和一些 Bean，例如 @Configuration、@Slf4j、@RequiredArgsConstructor 和 @Bean。

主要方法是 securityFilterChain()，它使用 HttpSecurity 对象来配置 Spring Security 的策略。在这个方法中，它使用 authorizeRequests() 方法来定义哪些请求需要被认证，使用 formLogin() 方法来配置表单登录，使用 logout() 方法来配置注销操作，使用 exceptionHandling() 方法来处理异常情况，使用 csrf() 方法来禁用 CSRF 保护，并添加了一个自定义的 TokenFilter 过滤器来验证请求中的令牌。

该类的主要作用是配置 Spring Security 的策略，包括认证、授权、登录、注销和异常处理等方面。PasswordEncoder Bean 用于加密密码，CustomAuthenticationSuccessHandler、CustomAuthenticationFailureHandler 和 CustomLogoutSuccessHandler 分别用于处理认证成功、认证失败和注销成功事件。AuthenticationEntryPointImpl 用于处理未经身份验证的请求，并返回错误响应。
```java
@Configuration
@Slf4j
@RequiredArgsConstructor
public class SecurityConfig {
  private final CustomAuthenticationSuccessHandler customAuthenticationSuccessHandler;
  private final CustomAuthenticationFailureHandler customAuthenticationFailureHandler;
  private final CustomLogoutSuccessHandler customLogoutSuccessHandler;
  private final TokenFilter tokenFilter;
  private final AuthenticationEntryPointImpl authenticationEntryPoint;

  @Bean
  public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
  }

  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {
    return httpSecurity
        .authorizeRequests()
        .anyRequest()
        .authenticated()
        .and()
        .formLogin()
        .successHandler(customAuthenticationSuccessHandler)
        .failureHandler(customAuthenticationFailureHandler)
        .and()
        .logout()
        .logoutUrl("/logout")
        .logoutSuccessHandler(customLogoutSuccessHandler)
        .and()
        .exceptionHandling()
        .authenticationEntryPoint(authenticationEntryPoint)
        .and()
        .csrf()
        .disable()
        .addFilterBefore(tokenFilter, UsernamePasswordAuthenticationFilter.class)
        .build();
  }
}

```

#### 加载用户数据到认证系统中

这是一个 Spring Security 的 UserDetailsService 实现类，用于从数据库中加载用户信息。该类使用了 Spring 的注解来声明依赖关系和一个构造函数，例如 @Component 和 @RequiredArgsConstructor。

主要方法是 loadUserByUsername()，它重写了 UserDetailsService 接口中的方法，并通过提供的用户名从数据库中加载用户信息。在这个方法中，它使用 AdminUserRepository 对象来查询数据库中的用户信息，并将其转换为 UserDetailsImpl 对象，然后返回 UserDetailsImpl 对象作为 Spring Security 中的 UserDetails 接口的实现。

该类的主要作用是从数据库中加载用户信息，并将其转换为 Spring Security 中的 UserDetails 接口的实现。它使用 AdminUserRepository 对象来查询数据库中的用户信息，并将其转换为 UserDetailsImpl 对象，该对象包含了用户的身份、权限和其他相关信息。在 Spring Security 中，UserDetailsService 接口用于从外部数据源（例如数据库）中加载用户信息，并将其提供给认证过程使用。

```java
@Component
@RequiredArgsConstructor
public class UserDetailServiceImpl implements UserDetailsService {
  private final AdminUserRepository adminUserRepository;

  /** 此接口重写了通过用户名加载用户的接口，采取读取数据库中的数据 **/
  @Override
  @Transactional(readOnly = true)
  public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    var adminUser =
        adminUserRepository
            .getByUsername(username)//这行是使用了@EntityGraph("AdminUser.all")在实体类上定义一个命名为AdminUser.all的实体关系图。这个图包括一个名为 "AdminUser.roles.permissions" 的子图，其中指定了 roles 关联的 permissions 属性应该与 roles 属性一起被预取。attributeNodes 元素还包括一个对 roles 属性的引用，并指定应该使用 "AdminUser.roles.permissions" 子图来预取 roles 关联的 permissions 属性。
            .orElseThrow(() -> new UsernameNotFoundException("用户不存在"));
    return new UserDetailsImpl(adminUser);
  }
}

```