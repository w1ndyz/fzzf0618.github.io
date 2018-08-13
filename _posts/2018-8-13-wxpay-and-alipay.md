---
layout: post
title:  “微信扫码支付以及支付宝扫码支付IN RAILS”
date:   2018-08-13 09:02:59
author: Chris Wang
categories: ror
tags: alipay wxpay 支付扫码 rails
---

### 引言
在工作上遇到了需要在后台集成`支付宝`和`微信`支付的扫码功能，在完成一系列的工作后，我觉得有必要总结和归纳一下。

### 开头
首先，我要介绍一个生成二维码的`Gem`包，名字是`rqrcode`，具体文档可以看[这里](https://github.com/whomwah/rqrcode)。这个gem包的主要作用是，在支付宝和微信支付回调返回二维码图片的链接时，将链接转化为base64的图片传给前端渲染。

### 支付宝扫码
进入[支付宝开放平台](https://open.alipay.com/platform/home.htm)， 登录账号->开发接入->支付应用->接入应用。选择接入的应用然后点击详情->应用信息->开发设置。在这里，我们设置支付的回调，加密方式等等。支付宝提供了很方便的沙箱服务，我们可以先在沙箱环境中调试支付应用，然后在部署到生产环境。支付宝[api文档](https://docs.open.alipay.com/200)可以在这里查看。在rails中，我接入了`Rei`写的gem[alipay](https://github.com/chloerei/alipay)，具体文档可以去看看。

接下来就是创建生成订单的接口和回调的接口
```ruby
#lib/alipay_qr.rb
class AlipayQR
  def self.create_order(options)
      # 建立一个客户端以便快速调用API
      @alipay_client = Alipay::Client.new(
        url: CONFIG.alipay_api_url, #在secret.yml中配置
        app_id: CONFIG.alipay_appid, #在secret.yml中配置
        app_private_key: CONFIG.alipay_app_private_key, #在secret.yml中配置
        alipay_public_key: CONFIG.alipay_app_public_key #在secret.yml中配置
      )
      response = @alipay_client.execute(
        method: 'alipay.trade.precreate',
        notify_url: CONFIG.alipay_callback_url,  #在secret.yml中配置
        biz_content: {
          out_trade_no: options[:order_id],
          total_amount: options[:fee]/100,
          subject: options[:plan_name],
        }.to_json(ascii_only: true), # to_json(ascii_only: true) is important!
        timestamp: Time.now.to_s(:db)
      )
      # 提取二维码地址
      qr_code_url = JSON.parse(response)["alipay_trade_precreate_response"]["qr_code"]
      # 返回二维码链接
      qr_code_url 
  end
end
```
接下来处理链接变成base64的图片
```ruby
@payment_url = Qrcode.build(qr_code_url)
```
```ruby
#lib/qrcode.rb
class Qrcode

  # 根据url生成二维码图片
  def self.build(url)
    return if url.blank?
    qr = RQRCode::QRCode.new(url)
    qr.as_png.resize(400, 400).to_data_url
  end

end
```
用户扫完码支付完成之后回调配置的路径，获取所需要的参数，并存入数据库即可。

### 微信扫码
微信的文档需要吐槽一下，看着没有支付宝的清晰，并且基本上每次所做的操作都要扫码授权登录等等。沙箱环境的搭建也没有支付宝成熟。这里是微信的[开发文档](https://pay.weixin.qq.com/wiki/doc/api/index.html)，如果有别的需求也可以上去看看。在rails中，我接入了`Jasl`写的gem[wx_pay](https://github.com/jasl/wx_pay)。微信接口的接入会稍微麻烦一点，包含了开发者审核，应用审核，安全证书的安装，apiclient_cert的下载等等，完成一系列的配置之后，再来看看代码。

首先我们在`initialize`文件夹下初始化微信的配置
```ruby
#initialize/wxpay.rb
WxPay.appid = CONFIG.wxpay_appid #在secret.yml中配置
WxPay.key = CONFIG.wxpay_appkey #在secret.yml中配置
WxPay.mch_id = CONFIG.wxpay_merchant_id #在secret.yml中配置

#下载的cert文件存放在settings文件夹下
filepath = File.expand_path('../../settings/apiclient_cert.p12', __FILE__)
WxPay.set_apiclient_by_pkcs12(File.read(filepath), WxPay.mch_id) if ENV['RAILS_ENV'] == 'production'

WxPay.extra_rest_client_options = { timeout: 2, open_timeout: 3 }
```
然后编写请求二维码的api
```ruby
#lib/wxpay_qr.rb
class WxpayQR
  
  def self.create_order(options)
    params = {
      body: options[:plan_name],
      out_trade_no: options[:order_id],
      total_fee: options[:fee],
      spbill_create_ip: options[:real_ip],
      notify_url: CONFIG.wxpay_notify_url, #微信异步操作callback
      trade_type: 'NATIVE', # could be "MWEB", ""JSAPI", "NATIVE" or "APP", 如果不是扫码，一般是"JSAPI"
    }
    r = WxPay::Service.invoke_unifiedorder params
    qrcode_url = r["code_url"]
    qrcode_url
  end

end
```
这时同样我们调用

```ruby
@payment_url = Qrcode.build(qr_code_url)
```
返回base64图片，代码同支付宝。同样的，在客户微信支付扫码完成之后，微信会回调`notify_url`，在callback中我们来取微信回传的值。这里需要注意的是，微信的回调和支付宝回调不一样，微信的回调是`xml`文件，在取的时候，文档也给了我们例子：
```ruby
def wxpay_callback
    result = Hash.from_xml(request.body.read)["xml"]
    if WxPay::Sign.verify?(result)
      #你需要处理的具体内容

      render plain: 'success', layout: false
    else
      render plain: 'fail', layout: false
    end
  end
```
#### 以上。
