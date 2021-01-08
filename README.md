# 自我 code review checklist

#### 1.对于弱类型语言，检查传递参数类型是否正确，多个参数时顺序是否正确
#### 2.命名是否准确直白的表达了方法的含义
#### 3.多思考空值（nil）可能的情况，可以做强制类型转换或者是 `try`、`&`
#### 4.检查是否有拼写错误，碰见 Bug 也首先检查是否拼写错误
#### 5.数字相除时，考虑分子或分母是零的情况，避免出现 Float::INFINITY
#### 6.在复杂数据结构（多维数组，嵌套哈希）中获取数据时，注意验证代码逻辑性，多写测试
#### 7.写完了 task 脚本要在提交 PR 时跑一遍，别代码部署好了，一跑就报错
#### 8.用 Module 做作为作用域时，检查调用的 class/module 是否会出现常量查找的问题，不确定的话就用 `module.nesting` 测试一下
  ```ruby
  module WxPub
    module Menu
      def self.fetch_all
      end
    end
  end

  module Admin
    module WxPub
      class ConditionalMenusController
        # Module.nesting 这里的常量查找树是 [Admin::WxPub::ConditionalMenusController, Admin::WxPub, Admin]，并没有 WxPub，虽然直觉看起来像是
        def menus
          WxPub::Menu.fetch_all # 这里要在顶级作用域查找 ::WxPub::Menu 或者 Object::WxPub::Menu
        end
      end
    end
  end

  Admin::WxPub::ConditionalMenusController.new.menus # => NameError: uninitialized constant Admin::WxPub::Menu
  ```
#### 9.更改名字后确定每一个变量都修改完毕，最好直接用编辑器的全局替换
#### 10.Git 解决冲突的时候要仔细
#### 11.不是必要的修改不要提交上去，每次只改跟自己任务关联的代码
#### 12.方法名，变量名的单复数检查
#### 13.大量数据更新可以用 each_slice + transaction 或者是 find_each + transaction，节省数据库连接的时间，但要考虑都报错rollback的可能，transaction rollback 是全 rollback
#### 14.少用 logger debug，多思考逻辑，仔细阅读代码，考虑数据的可能性，不要心急，节省时间
#### 15.如果使用 Rails [active_record_shards](https://github.com/zendesk/active_record_shards)，要记得 `on_all_shards` 是迭代器，如果有多个 shards，block 代码会执行多次
  ```ruby
  on_all_shards do
    send_welcome_emal
  end
  ```
#### 16.使用 `URI.encode` 时要注意 double encoding 的问题
  ```ruby
  # 如果是外来的输入，最好这样写
  def encode(uri_str)
    URI.encode(URI.decode(uri_str))
  end
  ```
#### 17. `URI::HTTP.build` 或者 `URI::HTTPS.build` 时，主要中文参数会报错，需要先 encode
#### 18. 对已有接口修改时，要确定修改影响范围，别的未跟你说的，要自己考虑一下，尤其是接口权限的修改
#### 19. 避免使用 `after_save`，因为 `after_save`是在一个 `transaction` 中可能会多次触发 `save`，优先使用 `after_commit`
#### 20. redis set hash 的时候记得 hash 要 to_json
#### 21. 如果数据库分片，并在 Action 层设置了分片策略，新的 Action 记得要注册分片策略
#### 22. 不要使用 `after_initialize` 初始化 `attribute`，会造成 `select(:id)` 后的链式方法调用出现问题: `attribute_missing`
#### 23. 数据需要考虑是否批量处理（思考可能的数据量大小）
#### 24. sidekiq 如果要使用复杂参数的话，不要使用 symbol_keys, 要使用 string_keys
  > The arguments you pass to perform_async must be composed of simple JSON datatypes: string, integer, float, boolean, null(nil), array and hash. This means you must not use ruby symbols as arguments. The Sidekiq client API uses JSON.dump to send the data to Redis. The Sidekiq server pulls that JSON data from Redis and uses JSON.load to convert the data back into Ruby types to pass to your perform method. Don't pass symbols, named parameters or complex Ruby objects (like Date or Time!) as those will not survive the dump/load round trip correctly.
```ruby
params = JSON.dump({email_address: "someemailaddress@gmail.com", something_else: "thing"})
JSON.load(params) #=> {"email_address"=>"someemailaddress@gmail.com", "something_else"=>"thing"}
```
#### 25.建议使用 `each_with_object` 而非 `inject`，`inject` 需要思考迭代对象的返回，一不小心就会出现 nil 的情况
```ruby
ticket_params.inject([]) do |change_attrs, (key, value)|
  change_attrs << key if attrs[key] != value // 这里可能会不执行，不执行的话下次迭代 change_attrs 就是 nil 了
end

ticket_params.each_with_object([]) do |(key, value), change_attrs|
  change_attrs << key if attrs[key] != value
end
```
#### 26. 注意 model callback 逻辑，尤其是在其他模块中的，下面的例子中 profile 的 username 会为 nil
```ruby
class User
  include Profilable
  
  before_create :set_name
  
  private
  
  def set_name
    self.name = "Aka"
  end
end

module Profilable
  extend ActiveSupport::Concern

  included do
    before_create :set_profile
  end
  
  private
  
  def set_profile
    Profile.find_or_create(username: self.name)
  end
end
```

#### 27. https://stackoverflow.com/questions/19064209/how-is-each-with-object-supposed-to-work
#### 28. https://ksylvest.com/posts/2017-08-23/eager-loading-polymorphic-associations-with-ruby-on-rails
#### 29. https://sites.google.com/site/gzhpwiki/home/guo-cheng-shi-jian/http-xie-yi-zhong-de-ge-zhong-zhang-du-xian-zhi-zong-jie
#### 30. 多个条件(&&)选择按计算量大小顺序写(小的在前头)优先断路
```ruby
display_signature => true or false, no need more compute

# bad
if signature.present? && display_signature
  render_signature
else
  # do something
end

# good
if display_signature && signature.present?
  render_signature
else
  # do something
end
```
