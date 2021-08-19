# php集成测试

## PHPCS
静态测试包含以下两个组件：
- phpcs —— 静态测试，编码规范检查
- phpcbf —— 自动修复编码规范
### 安装
```
composer require --dev squizlabs/php_codesniffer:~2.7
```
### 运行
```
./vendor/bin/phpcs -i  //查看已安装的规范  
程序路径 规范路径  
./vendor/bin/phpcs --standard=phpcs.xml  
./vendor/bin/phpcbf --standard=phpcs.xml  
```

## PHPMD
静态测试
代码冗余度及质量检查，包括以下规范检查：
- Clean Code Rules
- Code Size Rules
- Controversial Rules
- Design Rules
- Naming Rules
- Unused Code Rules
### 安装
```composer require --dev phpmd/phpmd:~2.5  ```
### 运行
程序路径 测试路径 报告格式 规则路径  
```./vendor/bin/phpmd app,tests text resources/rulesets/cleancode.xml,resources/rulesets  /codesize.xml,resources/rulesets/controversial.xml,resources/rulesets/design.xml,resources/rulesets  /naming.xml,resources/rulesets/unusedcode.xml ``` 

## PHPUNIT
动态测试
可以进行以下几种具体测试：
- 单元测试
- 集成测试
### 安装
```
composer require --dev phpunit/phpunit:~5.0
```

### 运行
```./vendor/bin/phpunit --configuration phpunit.xml  ```

临时关闭规则  
代码块上添加注解命令，phpmd规则举例  
```@SuppressWarnings(PHPMD[.规则]) ```