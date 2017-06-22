* 创建数据库
> `php app/console doctrine:database:create`

* 删除数据库
> `php app/console doctrine:database:drop --force`

* 生成单个实体
> `php app/console doctrine:generate:entity`
> 1. 输入:AppBundle:Product会创建一个Product的Entity
> 2. 输入要映要的字段名称:name
> 3. 输入字段类型(默认字符串):string
> 4. 输入字段长度(默认255):255
> 5. 输入字段是否允许为空(默许不能为空):false
> 6. 输入字段是否唯一(默认不为唯一值):false
> 7. 回车完成

* 创建|更新数据表(前提是有对应表的Entity)
> `php app/console doctrine:schema:update --force`

* 更新所有实体(参数`--no-backup`不生成备份文件)
> `php app/console doctrine:generate:entities AppBundle --no-backup`

* 清理缓存：
> `php app/console cache:clear`
> 生产环境加上`-e prod`参数：
> `php app/console cache:clear  -e prod`

* 重新生成 app/bootstrap.php.cache：
> `php ./vendor/sensio/distribution-bundle/Resources/bin/build_bootstrap.php`