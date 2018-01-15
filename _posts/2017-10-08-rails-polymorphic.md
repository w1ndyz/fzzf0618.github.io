---
layout: post
title:  “Rails Polymorphic 多态关联”
date:   2017-10-08 16:09:59
author: Chris Wang
categories: ror
tags: rails polymorphic 多态关联
---

## 引言
`Polymorphic`多态关联是我一直没怎么弄明白的关联方式。下面来一起研究研究。

假设我们有两个`model`，一个是`user`，另外一个是`company`。他们都有一个共同的内容就是`description`。这个时候，我们就可以用多态关联来创建它:
```
rails g scaffold company name website
rails g scaffold users first_name last_name email birth_date:date
rails g model note notable:references{polymorphic} description:text
```
那么通过polymorphic创建出来的`migrate`是什么样的呢？我们来看一下:
```
class CreateNotes < ActiveRecord::Migration[5.1]
  def change
    create_table :notes do |t|
      t.references :notable, polymorphic: true
      t.text :description

      t.timestamps
    end
  end
end
```
在`rake db:migrate`之后，`schema`文件中：
```
create_table "notes", force: :cascade do |t|
  t.string "notable_type"
  t.integer "notable_id"
  t.text "description"
  t.datetime "created_at", null: false
  t.datetime "updated_at", null: false
  t.index ["notable_type", "notable_id"], name: "index_notes_on_notable_type_and_notable_id"
end
```
我们可以看到，它创建了一个`notable_type`, 还有`notable_id`。并且还创建了他们分别对应的索引。

在对应的`model`文件中，我们要建立对应的关联:
```
note.rb

class Note < ApplicationRecord
  belongs_to :notable, polymorphic: true
end
```
```
user.rb

class User < ApplicationRecord
  has_many :notes, as: :notable
end
```

```
company.rb

class Company < ApplicationRecord
  has_many :notes, as: :notable
end
```

在创建路由的时候：
```
routes.rb

Rails.application.routes.draw do
  resources :companies do
    resources :notes, module: :companies
  end

  resources :users do
    resources :notes, module: :users
  end

  ...
end

```

如果我们需要创建`user`, `company`所对应的`note`，我们可以这样:
```
notes_controller.rb

class NotesController < ApplicationController
  def new
    @note = @notable.notes.new
  end

  def create
    @note = @notable.notes.new note_params
    @notable.save
    redirect_to @notable, notice: "Your note was successfully posted."
  end

  private

    def note_params
      params.require(:note).permit(:description)
    end
end
```

在另外两个`controller`中，创建`@notable`:
```
users/notes_controller.rb

class Users::NotesController < NotesController
  before_action :set_notable

  private

    def set_notable
      @notable = User.find(params[:user_id])
    end
end

companies/notes_controller.rb

class Companies::NotesController < NotesController
  before_action :set_notable
  
  def create
    # NOTIFY
    super #这里的super会继承NotesController里的create
  end

  private

    def set_notable
      @notable = Company.find(params[:company_id])
    end
end
```
在对应的`html`文件中：
```
users/show.html.erb

<%= render partial: "notes/notes", locals: {notable: @user} %>
<%= render partial: "notes/form", locals: {notable: @user} %>

companies/show.html.erb

<%= render partial: "notes/notes", locals: {notable: @company} %>
<%= render partial: "notes/form", locals: {notable: @company} %>
```
```
notes/_form.html.erb

<%= form_with(model: [notable, Note.new], local: true) do |form| %>
  <div class="field">
    <%= form.label :description %><br/>
    <%= form.text_area :description %>
  </div>
  <%= form.submit class: "btn btn-primary" %>
<% end %>

notes/_notes.html.erb

<h3>Notes</h3>
<% notable.notes.each do |note| %>
  <p>
    <hr>
    <%= note.content %>
    <em><%= time_ago_in_words note.created_at %></em>
  </p>
<% end %>
```

创建成功后，表结构为:

id | notable_type | notable_id | description
---|--- | --- | ---
1 | User | 1 | 创建了一个des
2 | Company | 1 | 公司的描述为xxx

这里的`notable_id`是对应到各个`model`中的`id`。

`polymorphic`多态关联，让表结构重复的地方更简单，让关联也更加清晰。
