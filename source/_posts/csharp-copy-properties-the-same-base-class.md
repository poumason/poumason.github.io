---
title: 如何讓不同的子類別同步父類別屬性值
draft: true
date: 2024-06-03 22:20:47
tags: 
  - C#
  - .NET
  - ASP.NET
---
之前在開發程式時，總會習慣為 data model 建立個 base class 方便把常用的 properties 包裝起來，或是做一些客製的處理。

隨著繼續的子類別變多時，有遇過幾次 `需要將某個子類別 copy 屬性至不同子類別`。

按照物件導向原理∶ **子類別無法互轉，或是先轉父類別再轉子類別**。如果要做到複雜屬性的目的，只能經過 Relction 的方式了。

## 作法說明
1. 確認二個子類別均繼承相同的父類別
1. 建立一個 Extension 並且定義使用的物件必須是父類別家族的人
1. 利用 Reflection 的方式將 Properties 取出 (由於二者均是相同的父類別，所以該有的屬性均會存在)
1. 範例程式
  - 先定義範例用的父類別/子類別
  ```
  namespace TodoApi.Models
  {
      public abstract class BaseInfo
      {
          public string UserName { get; set; }  
          public string Old { get; set; }
      }  
      public class Student : BaseInfo
      {
          public string School { get; set; }  
          public string Dept { get; set; }
      }  
      public class Employee : BaseInfo
      {
          public string Company { get; set; }
          public string Salary { get; set; }
      }
  }
  ```
  - Extension 寫法
  ```
  public static class ObjectExtensions
  {
      public static void CopyProperiesFromBaseClass<T, Y>(this T self, Y from)
      where T : BaseInfo
      where Y : BaseInfo
      {
          var selfProperties = self.GetType().GetProperties();
          var fromProperties = from.GetType().GetProperties();
          foreach (var item in fromProperties)
          {
              Console.WriteLine($"feed proprerty {item.Name}={item.GetValue(from)}");
              selfProperties.Where(x => x.Name == item.Name).FirstOrDefault()?.SetValue(self, item.GetValue(from));
          }
      }
  }
  ```
  - 用法(以 ASP.NET 為例)
  ```
  [HttpPost]
   public Employee ConvertData(Student student) {
       var sBaseInfo = student as BaseInfo;
       var employee = new Employee();   
       employee.CopyProperiesFromBaseClass(sBaseInfo);   
       return employee;
   } 
  ```

# References
- [Property Copying Between Two Objects using Reflection](https://www.pluralsight.com/resources/blog/guides/property-copying-between-two-objects-using-reflection)