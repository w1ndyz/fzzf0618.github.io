---
layout: post
title:  “近期工作所遇到的问题”
date:   2017-09-27 10:48:59
author: Chris Wang
categories: ror
tags: rails ruby facade rubocop
---

## 引言
这段时间做项目,完成度不是很好，但是从项目中学到了一些好的东西，总结一下。

* `Facade`

这么做的好处是简化了`model`和`controller`的代码。一旦到项目后期，就只需要在`Facade`下修改业务逻辑，比较清晰明了。
```
#controller层
module Admin
  class ProcessingFeedbacksController < AdminBaseController
    before_action :fetch_facade, only: %i[create index]
    def index
      @feedbacks = @processing_feedback_facade.index
    end

    def create
      if @processing_feedback_facade.save
        redirect_to admin_processing_feedbacks_path,
                    notice: 'Successfully created the feedback'
      else
        redirect_to_feedback
      end
    end

    private

    def fetch_facade
      @processing_feedback_facade = Admin::ProcessingFeedbackFacade.new(nil, params)
    end

    def redirect_to_feedback
      redirect_to admin_processing_feedbacks_path
    end
  end
end
```

可以从上面看到`processing_feedback_facade`就是它的`service`层。而`service`层的代码又是什么样的呢？

```
module Admin
  class ProcessingFeedbackFacade
    attr_reader :text, :params

    def initialize(text = nil, params = nil)
      @text = text || ProcessingFeedbackDefinition.new
      @params = params
    end

    def form
      @form ||= Admin::ProcessingFeedbackForm.new(text)
    end

    def save
      return form.save if form_valid?
      false
    end

    def index
      ProcessingFeedbackDefinition.where(active: true)
    end

    private

    def form_valid?
      return false unless feedback_param
      form.validate(feedback_param)
    end

    def feedback_param
      params[:admin_processing_feedback]
    end

    def processing_feedback
      @feedback ||= ProcessingFeedbackDefinition.find(params[:id])
    end
  end
end
```

### 具体来说
我们可以看到`model`是`ProcessingFeedbackDefinition`，这里面还出现了`ProcessingFeedbackForm`，这个主要是继承了`Reform::Form`的自定义表单，做了一些类似于表单验证的内容，具体就不说明了。在整个`Facade`中，写所有的业务逻辑，包括数据交互，表单提交，页面的操作等等。`Facade`是基于ROR的一种新的设计模式，意思是：可重用的面向对象的软件设计。`Facade`定义了一个高层接口，使子系统更容易使用，它是结构化设计模式的一种。

我们知道ROR是一款MVC框架，`View`层负责展示数据,`Model`层负责操作数据，`Controller`层负责将他们之间连接起来，但这会造成一个问题就是:会生成很多的代码在`View`层去展示出来,随着业务逻辑的增多，项目变得越来越大，`Model`层和`Controller`层的代码也会越来越多，这会很难管理和修改。举个例子，这是一个很寻常的`controller`
```
class UsersController < ApplicationController
  def index
    @user = User.new
    @last_active_users = User.active.order(created_at: :desc).limit(10)
    @vip_users_presenter = VipUsersPresenter.new(User.active.vip)
    @messages = current_user.messages
  end
end
```

你可以看到里面有很多的初始化和赋值，那我们用`Facade`的设计模式来解决会是什么样的呢？

```
# app/facades/users_facade.rb
class UsersFacade
  attr_reader :current_user, :vip_presenter

  def initialize(current_user, vip_presenter=VipUsersPresenter)
    @current_user = current_user
    @vip_presenter = vip_presenter
  end

  def new_user
    User.new
  end

  def last_active_users
    @last_active_users ||= active_users.order(created_at: :desc).limit(10)
  end

  def vip_users
    @vip_users ||= vip_presenter.new(active_users.vip).users
  end

  def messages
    @messages ||= current_user.messages
  end

  private
  def active_users
    User.active
  end
end
```
在`View`层里面,代码就变成了这样

```
<%= render @user_facade.last_active_users %>
<%= render @user_facade.messages %>
<%= render 'users/form', user: @user_facade.new_user %>
<%= render @user_facade.vip_users %>
```

可以看到，`View`和`Controller`的代码量很小，而你想修改什么的时候，直接去facade修改就行了。还有一个好处就是，在做测试的时候也很好写。但是它也会有一点缺点, 就是在你的项目中多抽象出来了一层。`Facade`是不可重用的，因为一个`Facade`是为一个`Controller`量身定制的。好处意味着，你可以统一用一个`Facade`面对`view`层的交互。

* 代码格式的校验

这里推荐一个Gem，名字是`rubocop`。它结合了一些ruby的最佳实践，在写完代码后，运行`rubocop -a`会检视目录下的所有文件，包括测试等等，如果不符合规范，有的会直接修改成最佳实践，有的会提醒你是否需要修改。

