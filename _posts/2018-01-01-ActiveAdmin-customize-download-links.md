---
layout: post
title:  “Rails ActiveAdmin Customize Download link”
date:   2018-01-01 23:29:59
author: Chris Wang
categories: ROR
tags: rails activeadmin down_links
---

## 引言
在工作中用了一段时间的activeadmin快速构建CRM。遇到了文档上不常有的Tip，在这里分享下。

自定义文件`download_links`
```
# config/initialize
config.namespace :admin do |admin|
    admin.download_links = [:xml, :pdf, :xxx]
end


module ActiveAdmin::ViewHelpers::DownloadFormatLinksHelper
  def build_download_format_links(formats = self.class.formats)
    params = request.query_parameters.except :format, :commit

    div class: "download_links" do
      span I18n.t('active_admin.download')
      formats.each do |format|
        if format == :aba
          a("Download XXX", href: xxx_download_path(params: params))
        else
          a format.upcase, href: url_for(params: params, format: format)
        end
      end
    end
  end
end

```