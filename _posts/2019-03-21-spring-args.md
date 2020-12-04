---
layout: post
title: Spring Boot - 读取参数
date: 2019-03-21
categories:
    - Spring
comments: true
permalink: spring-args.html
---

```
@SpringBootApplication
public class Application implements ApplicationRunner {

    private static final Logger logger = LoggerFactory.getLogger(Application.class);

    public static void main(String... args) throws Exception {
        SpringApplication.run(Application.class, args);
    }

    //java -jar microservice-argument-passing-0.0.1.jar this-is-a-non-option-arg --server.port=9090  --person.name=github.com --person.mail=mail@github.com
    @Override
    public void run(ApplicationArguments args) throws Exception {
        logger.info("Application started with command-line arguments: {}", Arrays.toString(args.getSourceArgs()));
        logger.info("NonOptionArgs: {}", args.getNonOptionArgs());
        logger.info("OptionNames: {}", args.getOptionNames());

        for (String name : args.getOptionNames()){
            logger.info("arg-" + name + "=" + args.getOptionValues(name));
        }

        boolean containsOption = args.containsOption("person.name");
        logger.info("Contains person.name: " + containsOption);
    }
}
```

输出

```
NonOptionArgs: [this-is-a-non-option-arg]
OptionNames: [server.port, person.name, person.mail]
arg-server.port=[9090]
arg-person.name=[github.com]
arg-person.mail=[mail@github.com]
Contains person.name: true
```


我们也可以直接注入一个`ApplicationArguments`对象

```
@Component
public class ArgService {
    private static final Logger logger = LoggerFactory.getLogger(ArgService.class);

    @Autowired
    private ApplicationArguments args;

    @PostConstruct
    public void init()  {
        logger.info("Application started with command-line arguments: {}", Arrays.toString(args.getSourceArgs()));
        logger.info("NonOptionArgs: {}", args.getNonOptionArgs());
        logger.info("OptionNames: {}", args.getOptionNames());

        for (String name : args.getOptionNames()){
            logger.info("arg-" + name + "=" + args.getOptionValues(name));
        }

        boolean containsOption = args.containsOption("person.name");
        logger.info("Contains person.name: " + containsOption);
    }
}
```