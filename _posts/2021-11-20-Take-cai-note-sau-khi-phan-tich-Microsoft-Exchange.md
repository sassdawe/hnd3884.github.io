---
title: Take cái note sau khi phân tích Microsoft Exchange
author: HoangND
date: 2021-11-30 15:00:00 +0700
categories: [PoC]
tags: [cve-2021-34520, microsoft sharepoint]
---

Ở bản vá tháng 11 của Exchange, một lỗ hổng có mã CVE-2021-42321 được Microsoft tức tốc cảnh báo người dùng phải nhanh chóng cập nhật bản vá để tránh bị khai thác. 
Về bài PoC chi tiết về quá trình diff code đã có [Blog của NCSC](https://blog.khonggianmang.vn/phan-tich-ban-va-thang-11-cua-microsoft-exchange/), ở đây mình chỉ bổ sung một số chi tiết
và bước cuối trong quá trình exploit thôi.

## User configuration
Như ở bài blog của NCSC, hàm gây ra lỗi là hàm __TryDeserialize__ của lớp __Microsoft.Exchange.Data.ApplicationLogic.Extension.OrgExtensionSerializer__

![image](https://user-images.githubusercontent.com/61985236/142719544-60184c07-dbb1-4449-bc5b-03e01653ab08.png)

Đặt debug ở hàm gây lỗi, sử dụng chức năng trên trang add-ins ở /ecp để quan sát biến userConfiguration

![image](https://user-images.githubusercontent.com/61985236/142719554-a1732a06-d4ea-4ac6-902e-eeb987bed141.png)

![image](https://user-images.githubusercontent.com/61985236/142719558-9b3c38be-155e-42a9-850e-1f948c64aa0f.png)

Quan sát thấy userConfiguration có thuộc tính ConfigurationName là **ExtensionMasterTable** và có dạng dictionary chứa 3 thuộc tính OrgDO, OrgExtV, OrgChkTm. 
Để hàm nhảy vào được đoạn deserialize thì tham số OrgDO của biến userConfiguration phải có giá trị là false, mặc định giá trị hiện tại đang là true.

Sau đấy cần tiếp tục tìm kiếm cách update user configuration, đoạn này ở trang [doc](https://docs.microsoft.com/en-us/exchange/client-developer/web-service-reference/updateuserconfiguration-operation) của Microsoft đã mô tả cực kỳ chi tiết, tuy nhiên lại có một chi tiết khá lú
mà mình sẽ nhắc đến ngay bên dưới 😂. Nói qua một chút Exchange có hai endpoind chính /ecp được dùng để quản lý các cài đặt, /owa để quản lý hộp thư /ews để giao tiếp với Exchange Server
thông qua SOAP. Ở tài liệu nhắc đến ở trên họ cũng đã cho mình một ví dụ hoàn chỉnh về SOAP request, cùng ngó qua một chút
  
```xml
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xmlns:m="https://schemas.microsoft.com/exchange/services/2006/messages"
               xmlns:t="https://schemas.microsoft.com/exchange/services/2006/types"
               xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
               xmlns:xs="http://www.w3.org/2001/XMLSchema">
  <soap:Header>
    <t:RequestServerVersion Version="Exchange2010" />
  </soap:Header>
  <soap:Body>
    <m:UpdateUserConfiguration>
      <m:UserConfiguration>
        <t:UserConfigurationName Name="TestConfig">
          <t:DistinguishedFolderId Id="drafts"/>
        </t:UserConfigurationName>
        <t:Dictionary>
          <t:DictionaryEntry>
            <t:DictionaryKey>
              <t:Type>String</t:Type>
              <t:Value>PhoneNumber</t:Value>
            </t:DictionaryKey>
            <t:DictionaryValue>
              <t:Type>String</t:Type>
              <t:Value>111-111-5555</t:Value>
            </t:DictionaryValue>
          </t:DictionaryEntry>
        </t:Dictionary>
      </m:UserConfiguration>
    </m:UpdateUserConfiguration>
  </soap:Body>
</soap:Envelope>
```
 
Từ ví dụ này có thể thấy một user configuration sẽ có các thành phần bắt buộc là Name cùng FolderId, bên dưới là các DictionaryEntry, cấu trúc này khớp với biến userConfiguration
mình đã quan sát ở trên. Như vậy mình đã biết các thành phần là Name và DictionaryEntry, phải có thêm FolderID mới có thể update được config này.



