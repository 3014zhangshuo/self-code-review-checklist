# 自我 code review checklist

* 对于弱类型语言，检查传递参数类型是否正确，多个参数时顺序是否正确
* 命名是否准确直白的表达了方法的含义
* 多思考空值可能的情况
* 检查是否拼错字，碰见BUG首先也要检查是否有错字
* 数据相除时，考虑分子 / 分母是零的情况，避免出现Float::INFINITY
* 在复杂数据结构（多维数组，嵌套哈希）中获取数据时，注意验证代码逻辑性，多写测试
* 写完了 task 脚本要在提交 PR 时跑一遍
* 用 Module 做作为作用域时，检查调用的 class/module 是否会出现常量查找的问题
* 更改名字后确定每一个变量都修改完毕，最好直接用编辑器的全局替换
* Git 解决冲突的时候要仔细
* 不是必要的修改不要提交上去，每次只改跟自己任务关联的代码
* 方法名，变量名的单复数检查
* 大量数据更新可以用 each_slice + transaction 或者是 find_each + transaction，节省数据库连接的时间，但要考虑都报错rollback的可能，transaction rollback 是全 rollback
* 少用 logger debug，多思考逻辑，仔细阅读代码，考虑数据的可能性，不要心急，节省时间
* 如果使用 Rails [active_record_shards](https://github.com/zendesk/active_record_shards)，要记得 `on_all_shards` 是迭代器，如果有多个 shards，block 代码会执行多次
