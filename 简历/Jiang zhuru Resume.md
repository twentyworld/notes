# Contact information
- Mobile: 13871530497
- Email:tyriontwenty@gmail.com
---

# Personal information
 - Jiang Zhuru / Miss / 1992
 - Bachelor / Computer Science, Shenyang Technology University
 - Working experience: 4 years
 - Expected position: Java Engineer
 - City of expectations: Hangzhou, Shanghai
---

# Working Experience

## Citigroup Software Technology and Services (China) Limited. (April 2018 ~ present)

### The Risk Administration Portfolio Information System
The global risk management system `RAPID` is used to collect-manage and maintain customer credit data, and send approved credit data to downstream of the business. Customer information is collected from multiple internal systems, and traders can view the unified view in the system interface.
- Responsible for designing a unified, easy-to-use, scalable data loading mechanism. Internal framework definition, the application does not directly access the persistence layer. Active data is loaded into the JVM cache at the beginning of the application load. Use `mybatis` to achieve separate loading/bulk loading of data. The loading process uses asynchronous multi-threading model and uses Hibernate JPA to implement data saving.
- Responsible for system performance optimization, changing the data to `Mybatis`(loading data from the original `JPA` before), and optimizing the index for query conditions. Analyze resource bottlenecks and change single-threaded operations to multi-threaded parallel operations to improve performance.
- Design service monitoring functions to monitor high-time business threads in real time.

## VanceInfo Technologies Inc. Expatriate Citigroup Software Technology and Services (China) Limited. (November 2015 ~ April 2018)

### Automated Test Framework (Global Test Framework)

The main function of the project is to simulate the operation of `QA` in the browser through visual drop steps, thus achieving the purpose of automated testing.
- Drop animations with `Swing` while using `Selenium` with browser action mapping.
- Implement `Debug` function to track execution at any point of operation.

The project solves the problem of `QA` without programming and the ability to draw scripts. Improve test efficiency and shorten test work time from ten working days to less than two working days.

### Risk Administration Portfolio Information System
Participate in the entire process of product testing and project development process including requirements design, design review, develop test plans, execute test cases, defect tracking and software quality analysis.
- Execute test cases using the linux system.
- Use junit for white box testing.
- Develop automated test tools to help with testing.

### Enterprise Project Continuous Delivery Platform
- Using `Bitbucket` to maintain the `code review` mechanism, the project is split into different `Bundle` modules, and there is a dependency between `Bundle` .
- Use `Teamcity` to maintain the compilation order and rules of each module. Accelerate multi-programmer parallel development efficiency; continuous compilation, automatic pre-test, immediately find code problems, prevent dirty code `Merge`, and ensure code quality.
- Design test department's unified test automation framework, greatly simplifying the difficulty of automated scripting.
- Automatically release updated versions to the deployment container `uDeploy`, which can be configured for various environments and deployed to the production environment with one click.
---

##technology sharing
- In-company sharing session in September 2017: Golbal Test Framework User Guide
- In-company sharing session in April 2017: Enterprise project continuous delivery platform design concept
