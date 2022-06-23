---
title: 模板方法
comments: true
---

今天要学习的是模板方法模式，结合实际思考一下，我们一般会在什么时候需要用到模板，偷懒的时候用的可能比较多。比如说我们要准备一个技术分享会，问问同事有没有模板，省的从零开始。还有就是转正答辩，找小伙伴要个模板参考下，但是工作内容还是要填写我们自己干了啥。

<!--more-->

## 模板方法

使用场景就是，执行某些特定步骤的操作是相同的，但是你可以通过不同的方式实现。这种情况可以考虑模板方法模式。模板方法模式是通过把不变行为搬移到超类，去除子类中的重复代码来体现它的优势。

> 我怎么感觉学来学去，这几种模式也太相似了，这种使用场景通过策略模式是不是也行？你完成某个任务的步骤是一样的，但你是可以通过不同的方法..

众所周知，Golang 中是没有继承的，强行套出来的代码并不优雅。但是，不妨碍我们理解它。



假设产品给我们提需求，在一天内完成验证码生成工具，然后发送给客户，大致分成以下几个步骤：

- 生成随机的密码
- 保存这个密码到缓存中，方便后续校验
- 准备发送给用户的内容
  - 比如：尊敬的xx，您的验证码为xx，有效期为xx。
- 发送上述内容
- 显示发送是否成功



在发送的这一步，我们可以使用短信、邮箱的方式进行发送。



**策略模式重点指的是不同的算法，模板方法强调的是实现的方式，不管哪种方式实现的最终结果是一样的。但是策略模式不同，不同的策略产生的结果不一定相同。**

（自己总结的，仅供参考。）

代码来源：[golang-by-example](https://golangbyexample.com/template-method-design-pattern-golang/)

```go
// golang 代码实现
package main

import "fmt"

type iOtp interface {
    genRandomOTP(int) string
    saveOTPCache(string)
    getMessage(string) string
    sendNotification(string) error
    publishMetric()
}

type otp struct {
    iOtp iOtp
}

func (o *otp) genAndSendOTP(otpLength int) error {
    otp := o.iOtp.genRandomOTP(otpLength)
    o.iOtp.saveOTPCache(otp)
    message := o.iOtp.getMessage(otp)
    err := o.iOtp.sendNotification(message)
    if err != nil {
        return err
    }
    o.iOtp.publishMetric()
    return nil
}

type sms struct {
    otp
}

func (s *sms) genRandomOTP(len int) string {
    randomOTP := "1234"
    fmt.Printf("SMS: generating random otp %s\n", randomOTP)
    return randomOTP
}

func (s *sms) saveOTPCache(otp string) {
    fmt.Printf("SMS: saving otp: %s to cache\n", otp)
}

func (s *sms) getMessage(otp string) string {
    return "SMS OTP for login is " + otp
}

func (s *sms) sendNotification(message string) error {
    fmt.Printf("SMS: sending sms: %s\n", message)
    return nil
}

func (s *sms) publishMetric() {
    fmt.Printf("SMS: publishing metrics\n")
}

type email struct {
    otp
}

func (s *email) genRandomOTP(len int) string {
    randomOTP := "1234"
    fmt.Printf("EMAIL: generating random otp %s\n", randomOTP)
    return randomOTP
}

func (s *email) saveOTPCache(otp string) {
    fmt.Printf("EMAIL: saving otp: %s to cache\n", otp)
}

func (s *email) getMessage(otp string) string {
    return "EMAIL OTP for login is " + otp
}

func (s *email) sendNotification(message string) error {
    fmt.Printf("EMAIL: sending email: %s\n", message)
    return nil
}

func (s *email) publishMetric() {
    fmt.Printf("EMAIL: publishing metrics\n")
}

func main() {
    smsOTP := &sms{}
    o := otp{
        iOtp: smsOTP,
    }
    o.genAndSendOTP(4)
    fmt.Println("")
    emailOTP := &email{}
    o = otp{
        iOtp: emailOTP,
    }
    o.genAndSendOTP(4)
}
```
