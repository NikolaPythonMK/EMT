# EMT


1. FetchType(EAGER & LAZY) and EntityGraph
2. Jpa Projections
3. Locking (Optimistic & Pessimistic)
4. Views & Materialized Views
5. Scheduling
6. Application Events / Custom Events

============================================================================================================================================================
> FetchingType(EAGER & LAZY) & EntityGraph

    @OneToMany(mappedBy = "user", fetch = FetchType.EAGER)
    private List<ShoppingCart> carts;

    @EntityGraph(type = EntityGraph.EntityGraphType.FETCH, attributePaths = {"carts"})
    @Query("select u from User u")
    List<User> fetchAll();

    @EntityGraph(type = EntityGraph.EntityGraphType.LOAD, attributePaths = {"carts", "discount"})
    @Query("select u from User u")
    List<User> loadAll();

============================================================================================================================================================
> Jpa Projections

    @Query("select u.username, u.name, u.surname from User u") // will work without the querty too.
    List<UserProjection> takeUsernameAndNameAndSurnameByProjection();


model -> projections

public interface UserProjection {
    String getUsername();
    String getName();
    String getSurname();
}

============================================================================================================================================================
> Optimistic Locking

@Version
private Long version;

============================================================================================================================================================
> View
-virtual tables
-defined by a query
-combined data from multiple tables (join)
-the result is not stored in disk

Materialized view
-updated table when new data record is inserted/updated
scheduling -> periodicno da go ubdejtira

model -> views

@Data
@Entity
@Subselect("SELECT * FROM public.products_per_category") //Maps an immutable and read-only entity to a given SQL select expression.
@Immutable
public class ProductsPerCategoryView {

    @Id
    @Column(name = "category_id")
    private Long categoryId;

    @Column(name = "num_products")
    private Integer numProducts;
}

repositroty -> views

@Repository
public interface ProductsPerCategoryViewRepository extends JpaRepository<ProductsPerCategoryView, Long> {
}


> Materialized View
@Data
@Entity
@Subselect("SELECT * FROM public.products_per_manufacturers")
@Immutable
public class ProductsPerManufacturerView {

    @Id
    @Column(name = "manufacturer_id")
    private Long manufacturerId;

    @Column(name = "num_products")
    private int numProducts;
}

@Repository
public interface ProductsPerManufacturerViewRepository extends JpaRepository<ProductsPerManufacturerView, Long> {

    @Transactional
    @Modifying(clearAutomatically = true)
    @Query(value = "REFRESH MATERIALIZED VIEW public.products_per_manufacturers", nativeQuery = true)
    void refreshMaterializedView();
}

Service -> ProductService

void refreshMaterializedView();

Service -> Impl -> ProductServiceImpl

use the method refreshMaterializedView() on save() and update()

Yes, the @Subselect annotation is necessary to map a database view to an entity in JPA (Java Persistence API). The @Subselect annotation specifies the SQL query that should be used to retrieve data for the entity. In this case, the SQL query retrieves data from the public.products_per_manufacturers view.

Without the @Subselect annotation, JPA would assume that the entity corresponds to a table with the same name as the entity class (ProductsPerManufacturerView), and it would generate SQL queries based on that assumption. But since the entity actually corresponds to a view, those queries would not work as expected.

So, to correctly map a database view to an entity in JPA, you need to use the @Subselect annotation to specify the SQL query that retrieves data from the view.


============================================================================================================================================================
> Scheduling
1. Fixed delay
2. Cron expressions - https://docs.oracle.com/cd/E12058_01/doc/doc.1014/e12030/cron_expressions.htm

@SpringBootApplication
@EnableScheduling
public class EshopApplication {...


@Component
public class ScheduledTasks {

    private final ProductService productService;

    public ScheduledTasks(ProductService productService) {
        this.productService = productService;
    }

    @Scheduled(fixedDelay = 5000)
    public void refreshMaterializedView(){
        this.productService.refreshMaterializedView();
    }

}

============================================================================================================================================================
> Application Events / Custom Events

@Getter
public class ProductCreatedEvent extends ApplicationEvent {

    private LocalDateTime when;

    public ProductCreatedEvent(Product source) {
        super(source);
        this.when = LocalDateTime.now();
    }

    public ProductCreatedEvent(Product source, LocalDateTime when) {
        super(source);
        this.when = when;
    }
}



    private final ApplicationEventPublisher applicationEventPublisher;  (Dependency injection via constructor)

    @Override
    @Transactional
    public Optional<Product> save(String name, Double price, Integer quantity, Long categoryId, Long manufacturerId) {
        Category category = this.categoryRepository.findById(categoryId)
                .orElseThrow(() -> new CategoryNotFoundException(categoryId));
        Manufacturer manufacturer = this.manufacturerRepository.findById(manufacturerId)
                .orElseThrow(() -> new ManufacturerNotFoundException(manufacturerId));

        this.productRepository.deleteByName(name);
        Product product = new Product(name, price, quantity, category, manufacturer);
        this.productRepository.save(product);
        //this.refreshMaterializedView();

        this.applicationEventPublisher.publishEvent(new ProductCreatedEvent(product));
        return Optional.of(product);
    }


@Component
public class ProductEventHandlers {

    private final ProductService productService;

    public ProductEventHandlers(ProductService productService) {
        this.productService = productService;
    }

    @EventListener
    public void onProductCreated(ProductCreatedEvent event) {
        this.productService.refreshMaterializedView();
    }
}
