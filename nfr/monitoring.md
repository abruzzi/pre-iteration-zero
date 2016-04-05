## 监控

-  应用状态
-  系统状态

### Metrics

```groovy
dependencies {
    compile 'org.springframework:spring-context:4.2.4.RELEASE'
    compile 'org.springframework.data:spring-data-mongodb:1.8.4.RELEASE'
    
    compile 'io.dropwizard.metrics:metrics-core:3.1.2'
    compile 'io.dropwizard.metrics:metrics-jvm:3.1.2'
    compile 'io.dropwizard.metrics:metrics-graphite:3.1.2'
    compile 'com.ryantenney.metrics:metrics-spring:3.1.3'
}
```

```java
@Configuration
@EnableMetrics
public class MetricsConfig extends MetricsConfigurerAdapter {

    @Override
    public void configureReporters(MetricRegistry metricRegistry) {
        Graphite graphite = new Graphite(new InetSocketAddress("192.168.99.100", 2003));
        GraphiteReporter graphiteReporter = GraphiteReporter.forRegistry(metricRegistry)
                .prefixedWith("juntao-laptop")
                .convertRatesTo(TimeUnit.SECONDS)
                .convertDurationsTo(TimeUnit.MILLISECONDS)
                .filter(MetricFilter.ALL)
                .build(graphite);

        registerReporter(graphiteReporter);
        graphiteReporter.start(1, TimeUnit.MINUTES);


        metricRegistry.registerAll(new MemoryUsageGaugeSet());
        metricRegistry.registerAll(new ThreadStatesGaugeSet());
    }

}
```

```java
@RestController
public class PersonController {
    @Autowired
    private PersonRepository personRepository;

    @Metered(absolute = true, name = "metered.people.get.all")
    @Timed(absolute = true, name = "people.get.all")
    @RequestMapping(value ="/people", method = RequestMethod.GET)
    public List<Person> findAll() {
        return personRepository.findAll();
    }
}
```

### Graphite

### Grafana

### collectd
