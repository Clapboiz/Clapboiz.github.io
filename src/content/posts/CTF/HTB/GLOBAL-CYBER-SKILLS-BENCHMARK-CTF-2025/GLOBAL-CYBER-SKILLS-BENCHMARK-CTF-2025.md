---
title: "Global Cyber Skills Benchmark CTF 2025: Operation Blackout"
published: 2025-05-27
description: "Writeup CTF HTB"
image: ''
tags: ["CTF", "HTB", "CLOUD", "ML"]
category: Writeups
draft: false
---

# ML
## Uplink Artifact
![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/1.png)

Đọc vào đề và file data thì ta có thể biết được đây khả năng là nó bắt chúng ta tìm outlier hoặc cũng có thể là nó hide trong ảnh, bla bla, rất nhiều ngữ cảnh mà ta đặt ra.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/2.png)

Đây là dataset mà HTB cung cấp, vì thế điều đầu tiên nảy ra trong đầu khi làm với ML/AI thì ta phải xem được data đó nó có gì, những điểm nào đặc biệt, null, hay là số lượng mỗi lớp, ... Rất nhiều thứ mà chúng ta cần phải xem, bởi vì nếu chúng ta cứ cắm đầu training mà không xem thì khả năng rất cao mô hình đó fail.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/3.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/4.png)

Cũng như thường lệ, tôi sẽ check các thông tin của datasets, để có cái nhìn overview nhất.

thì sau khi tôi check thì các tỉ lệ như trên hình, bạn đọc và cũng có thể thấy. Và tôi thấy được rằng có điểm bất thường giữa `x, y` so với `z`. là `x, y` bạn có thể thấy được là giao động từ `0-25`, còn `z` sẽ từ `0-1`.

Tuy vậy nhưng tôi vẫn training :"))

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/5.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/6.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/7.png)

Đúng như dự đoán :")), Dù có train bằng ML/DL thì nó cũng đều rất tệ.

Lúc này tôi nghĩ đến phải chuẩn hóa nó về 1 khoảng giá trị nhất định giữa các cột, tức là phải đồng nhất về giá trị chứ không phải là cột cao cột thấp.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/8.png)

Sau khi chuẩn hóa thì nó vẫn thấp :")) thôi thì các cách liên quan đến train model, Anomaly Detection / Outlier đã phá sản rồi :")).

Ngồi nhìn lại thì lấy chỉ còn cách ảnh thôi. Như bạn được biết thì 1 bức ảnh có thể có 2 chiều hoặc 3 chiều. thì họ cho mình 3 cột tương ứng với trục x y z trong không gian 3 chiều và nó cũng sẽ tạo ra được ảnh, vì vậy ta sẽ dùng ý tưởng tạo ảnh nhé. Ảnh 2 hoặc 3 chiều nó sẽ tương ứng với trắng đen (2 chiều) hoặc rgb (3 chiều).

Ta có thể nhận thấy được rằng nếu theo logic đó thì x, y nó là tọa độ, còn z khả năng nó sẽ là màu.

Thì ý tưởng sẽ là

Đọc file CSV và ép kiểu x, y sang số nguyên để làm tọa độ pixel trong ảnh.

Chuẩn hóa giá trị z về khoảng 0-255 để biểu diễn màu (0-255 là kiến thức cơ bản về màu).

thì ý tưởng code sẽ là vậy. Nhưng nếu tôi làm theo ảnh 3 chiều thì nó sẽ phức tạp hơn nhiều vì z chỉ là màu sắc nên ta có thể bỏ, bởi vì nếu ý tưởng này thì chỉ cần trắng đen thì ta đã thấy text, nên  ta sẽ dùng ảnh 2 chiều thôi.

### Cách 1: Ảnh 2 chiều
```
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

df = pd.read_csv(r'D:\Users\Downloads\ml_uplink_artifact\uplink_spatial_auth.csv')

df['x'] = df['x'].astype(int)
df['y'] = df['y'].astype(int)

w = df['x'].max() + 1
h = df['y'].max() + 1

img = np.ones((h, w), dtype=np.uint8) * 255

# Mapping following label
for _, row in df.iterrows():
    x, y, label = int(row['x']), int(row['y']), row['label']
    if label == 1:
        img[y, x] = 0
    elif label == 2:
        img[y, x] = 150
    elif label == 3:
        img[y, x] = 200

plt.imshow(img, cmap='gray', origin='lower')
plt.axis('off')
plt.show()
```

ý tưởng thì giống bên trên, chỉ có ép kiểu x, y và mapping label -> màu thôi.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/9.png)

tạo ảnh thành công thì tôi lúc này khá hấp dẫn đến tôi lôi máy ra scan QR luôn :")). Và tất nhiên nó sẽ k đc.

Sau khi xem xét kĩ lại thì tôi thấy nó khả năng là 1 QR code nhưng định dạng lại không giống thì liệu cái label kia có nhiễu không, cũng giống như cột z chỉ để biểu thị màu sắc nên có thể bỏ, thì liệu cột z có label nhiễu ko??

Tôi tiếp tục giảm bớt, chỉ giảm còn 2 label trắng và đen, không còn xám hay nhạt j hết.

tiếp tục với ý tưởng đó thì ta sẽ cắt bớt và tạo thành ảnh khác chỉ có trắng và đen.

```
from PIL import Image
Image.fromarray(img).save(r"D:\Users\Downloads\ml_uplink_artifact\qr_code.png")
img1 = np.ones((h, w), dtype=np.uint8) * 255
for _, row in df.iterrows():
    x, y, label = int(row['x']), int(row['y']), row['label']
    if label == 1:
        img1[y, x] = 0  # đen

plt.imshow(img1, cmap='gray', origin='lower')
plt.axis('off')
plt.show()
```

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/10.png)

Ngon :")).

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/11.png)

Thành công

### Cách 2: Ảnh 3 chiều
```
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from PIL import Image

df = pd.read_csv(r'D:\Users\Downloads\ml_uplink_artifact\uplink_spatial_auth.csv')

df['x'] = df['x'].astype(int)
df['y'] = df['y'].astype(int)

z_min, z_max = df['z'].min(), df['z'].max()
df['z_norm'] = ((df['z'] - z_min) / (z_max - z_min) * 255).astype(np.uint8)

w = df['x'].max() + 1
h = df['y'].max() + 1

img_rgb = np.ones((h, w, 3), dtype=np.uint8) * 255

for _, row in df.iterrows():
    x, y = int(row['x']), int(row['y'])
    z_val = row['z_norm']
    img_rgb[y, x] = [255 - z_val, 255 - z_val, z_val]  # RGB

plt.imshow(img_rgb, origin='lower')
plt.axis('off')
plt.show()

Image.fromarray(img_rgb).save(r"D:\Users\Downloads\ml_uplink_artifact\rgb_visual.png")


img1 = np.ones((h, w), dtype=np.uint8) * 255

for _, row in df.iterrows():
    x, y, label = int(row['x']), int(row['y']), int(row['label'])
    if label == 1:
        img1[y, x] = 0 

plt.imshow(img1, cmap='gray', origin='lower')
plt.axis('off')
plt.show()

img_qr_like = Image.fromarray(img1)
qr_path = r"D:\Users\Downloads\ml_uplink_artifact\qr.png"
img_qr_like.save(qr_path)
```

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/12.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Uplink-Artifact/13.png)

khi scan sẽ ra flag.
## Decision Gate
![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Decision-Gate/1.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Decision-Gate/2.png)

Tiến hành vào check xem có thông tin gì không nhé.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Decision-Gate/3.png)

Tiến hành load model lên và predict theo example file.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Decision-Gate/4.png)

Vì model đã được đóng gói nên ta không thể biết được đây là model gì, ta check xem thì biết được model sử dụng là Decision Tree. và các classs, trong đó ta có thể thấy có 1 class tên khá đặc biệt `UNLOCK_FLAG_PATH`.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Decision-Gate/5.png)

Decision Tree hoạt động bằng cách duyệt từ nút gốc xuống các nút lá thông qua if/else

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Decision-Gate/6.png)

tôi sử dụng sklearn.tree.export_text() để trích xuất cây quyết định dưới dạng văn bản, và biết được chỉ có một nhánh duy nhất dẫn đến UNLOCK_FLAG_PATH đó là khi `feature_1 <= -5000002.47`

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Decision-Gate/7.png)

Như example input mà tác giả cung cấp thì tôi đưa 1 input vector 5 chiều và tôi sử dụng `-1e10` cho `feature_1`, đảm bảo nó nhỏ hơn `-5000002.47`. Và đúng như dự đoán nó dự đoán đúng class.

Tiến hành nc trên server và nhập payload thôi.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/ML/Decision-Gate/8.png)

Vậy là xong challenge này 

# CLOUD
## Dashboarded
![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/1.png)

Tiến hành truy cập vào ip đó xem sao.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/2.png)

Test sơ qua chức năng và đẩy qua burp thôi. Đẩy qua burp thì request/response của nó như sau

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/3.png)

nhìn url thì đầu tôi nghĩ đến case ssrf.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/4.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/5.png)

Đúng thật là nó bị ssrf.

Tiếp theo dựa vào kinh nghiệm nghịch cloud của tôi thì `http://169.254.169.254` là địa chỉ IP đặc biệt được sử dụng trong dịch vụ Metadata của AWS, cụ thể là EC2 Instance Metadata Service (IMDS).

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/6.png)

Đấy, confirm lại bằng google nhé.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/7.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/8.png)

Tiếp tục truy cập vào, và sau khi đến được meta-data

Cũng dùng tiếp cách này thì ta có thể lấy được các api key

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/9.png)

Đây là AWS temporary credentials cấp bởi Instance Metadata Service cho EC2 instance, thông qua một IAM Role có tên: APICallerRole

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/10.png)

Đây là API Key tạm thời dành cho EC2 instance, dùng để gọi AWS API nếu quyền được cấp.

### So sánh IAM Role vs EC2 Instance Identity Credentials

| Thuộc tính            | IAM Role (`APICallerRole`)                 | EC2 Instance Identity Credentials         |
|------------------------|--------------------------------------------|-------------------------------------------|
| Nguồn                  | iam/security-credentials/RoleName         | identity-credentials/ec2/security-credentials/ec2-instance |
| Cách cấp               | Gắn role trực tiếp                        | Identity Federation nội bộ                |
| Quyền truy cập         | Thường rộng hơn                           | Thường giới hạn                            |

-> IAM Role (`APICallerRole`) thường có nhiều quyền hơn và nên được ưu tiên khai thác trước.

Thì khi mà có key, thì điều đầu tiên tôi nghĩ đến khi mà tiếp xúc nhiều với cloud là set config, credential

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/11.png)

sau khi set up xong thì ta sử dụng cli thôi.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/12.png)

Check xem tôi đang dùng đúng danh tính và có quyền tối thiểu.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/13.png)

Khi tôi làm đúng nhưng nó vẫn không có quyền :>. tôi đã thử rất nhiều lệnh khác nhau liên quan đến các hành dộng và service của AWS nhưng vẫn k được, thì tôi phải dùng qua tool `enumerate-iam`, một bộ script Python được thiết kế đặc biệt để giúp kiểm tra và liệt kê các quyền hiện có của credentials AWS.

Điểm mạnh của enumerate-iam nằm ở khả năng tự động thử hàng loạt các hành động phổ biến trên nhiều dịch vụ AWS khác nhau. Thay vì phải thử từng lệnh thủ công và chờ đợi kết quả, công cụ này sẽ nhanh chóng phát hiện chính xác những quyền mà access key và role hiện tại được phép sử dụng.

Tôi đã chạy enumerate-iam với access key, secret key cùng session token của phiên làm việc hiện tại.kết quả là dynamodb.describe_endpoints() đã được phép thực thi thành công, trong khi một số hành động khác như globalaccelerator.describe_accelerator_attributes hay codedeploy.get_deployment_target thì bị từ chối và được loại bỏ.

Nhìn chung các role này chỉ có quyền hạn rất hạn chế, chủ yếu truy cập được một vài thao tác đọc nhẹ trên DynamoDB và STS, còn hầu hết các dịch vụ khác (như CodeDeploy hay Global Accelerator) đều bị chặn.

Hmm, cách này không khả thi rồi, tiến hành tìm cách khác thôi.

Chợt nhớ lại lúc đầu khi ta thao tác với UI thì có request này 

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/14.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/15.png)

Truy cập vào url của request này thì ta thấy như này. Tiếp tục tôi ngứa tay back lại 1 folder xem có gì hot không.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/16.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/17.png)

Giờ làm sao nhỉ ?? Để ý thì có thể thấy được rằng đây là 1 url liên quan đến 1 service của AWS.

Ta sử dụng tool awscurl, thì nó là một công cụ dòng lệnh giống như curl, nhưng có khả năng ký các HTTP request theo chuẩn AWS Signature Version 4 (SigV4), chuẩn mà các dịch vụ AWS (như API Gateway) yêu cầu để xác thực người dùng.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/18.png)

Đó bạn có thể hiểu đc chuẩn của AWS.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/19.png)

Một kiến thúc cũng liên quan đến sigV4 (chỉ đóng vai trò bổ sung thêm kiến thức), nó dùng để gửi các thông tin ký trong URL, như bạn đưa ra: X-Amz-Algorithm, X-Amz-Credential, X-Amz-Signature...	Thường dùng khi chia sẻ link truy cập tạm thời, ví dụ như file PDF trên S3

Vì API Gateway endpoint yêu cầu header-based SigV4 chứ không chấp nhận pre-signed URL, bạn không thể dùng cách chèn các X-Amz-* vào URL mà phải dùng công cụ như awscurl hoặc curl + ký tay.

Cài đặt:

```
pip install awscli awscurl
```

Ta tiếp tục dùng tool

```
awscurl \
  --service execute-api \
  --region us-east-2 \
  -X GET \
  --access_key ASIARHJJMXKM2JNGOY2Y \
  --secret_key <SECRET> \
  --security_token <SESSION_TOKEN> \
  https://inyunqef0e.execute-api.us-east-2.amazonaws.com/api/private
```

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/20.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Dashboarded/21.png)

Thì đã ra được flag.
## Vault
![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Vault/1.png)

Cũng vào url xem có gì hot.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Vault/2.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Vault/3.png)

Sau khi vào trang chủ thì ta có thể thấy 1 vài api, và cứ tiếp tục thực hiện chức năng thì ta được một vài api như trên.

khi mà tôi nhìn đến filename kia là 1 cái file thì tôi nghĩ ngay đến file inclusion.

Và api file kia thì tôi để ý output có api với các param như classification, id và nó đều có quy luật. Vì thế tôi liền nghĩ đến `?id=`, `?classification=` thì id tôi sẽ bruteforce, còn classification thì tôi sẽ đoán `PRIVATE, ...` nhưng nó vẫn k trả các file khác. và khi tôi thử các param như `?abc` hoặc 1 param nào đó tôi chắc chắn nó k tồn tại như `clapboiz` thì nó vẫn trả về full thì tôi nghĩ là bug nó không nằm ở chức năng này.

Vậy thì tìm attack surface khác thôi.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Vault/4.png)

Bạn có thể để ý là có `https://volnaya-vault.s3.eu-north-1.amazonaws.com` thì trong url này bạn có thể biết được là S3 bucket là `volnaya-vault`.

Ba URL sau đây là 3 cách khác nhau để truy cập vào một Amazon S3 bucket, cụ thể là volnaya-vault. Chúng khác nhau ở cách định dạng URL, nhưng bản chất đều trỏ về cùng một S3 bucket nếu nó tồn tại và được public hoặc bạn có quyền truy cập. Còn bạn muốn biết tại sao lại như này thì tham khảo link ở phía dưới của AWS để rõ nhé.

```
https://volnaya-vault.s3.eu-north-1.amazonaws.com/
https://s3.eu-north-1.amazonaws.com/volnaya-vault/
https://volnaya-vault.s3.amazonaws.com/
```

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Vault/5.png)

Sau khi vào thì ta thấy được rằng nó lộ các file như trên.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Vault/6.png)

Tiến hành vào thử response url của file này để tải nó về.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Vault/7.png)

À, nó đang ở thư mục `/vault/public` tức là hiện tại nó đang ở `vault/public/vault/private`. Ta cần dùng path traversal để back về 2 thư mục cho đúng với thư mục mà file này đang ở (chính là `vault/private`).

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Vault/8.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Vault/9.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/Vault/10.png)

Vậy là thành công lấy được flag.
## EBS
![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/1.png)

Đọc vào thì có thể thấy có CloudFront. Thì sơ qua CloudFront là một dịch vụ phân phối nội dung (CDN) của AWS. Nó giúp tăng tốc độ tải nội dung như hình ảnh, video, file tĩnh (HTML, CSS, JS), và thậm chí là API, bằng cách phân phối nội dung từ các máy chủ gần người dùng nhất trên toàn cầu. Đại loại là CloudFront giúp website nhanh hơn, an toàn hơn và tiết kiệm chi phí hơn. Nếu không dùng, bạn phải tự xử lý tất cả về tốc độ, cache, bảo mật, và mở rộng hệ thống. Ngoài ra có một ưu điểm mà nó được dùng nữa là IP hoặc DNS thật của EC2/S3 dễ bị lộ nếu không cấu hình tốt. Dẫn đến nên dùng CloudFront để che dấu IP

Tiếp theo vào url mà challenge cung cấp thôi.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/2.png)

```
https://ebs-we7uo726.auth.eu-north-1.amazoncognito.com/login?redirect_uri=https%3A%2F%2Fd25vt11c4zrsnp.cloudfront.net%2Fcallback&response_type=code&client_id=1s3s6r0e517r5h7m00ab0t58ra&identity_provider=COGNITO&scope=email%20openid&state=BfnUwyLLNJcbeMmUAMunYL4gsdacXIl5&code_challenge=z-4CUgID6OVWNNoXfxYPQ8hBln0rq_plitk5aJd7fSM&code_challenge_method=S256
```

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/3.png)

```
https://ebs-we7uo726.auth.eu-north-1.amazoncognito.com/signup?redirect_uri=https%3A%2F%2Fd25vt11c4zrsnp.cloudfront.net%2Fcallback&response_type=code&client_id=1s3s6r0e517r5h7m00ab0t58ra&identity_provider=COGNITO&scope=email%20openid&state=BfnUwyLLNJcbeMmUAMunYL4gsdacXIl5&code_challenge=z-4CUgID6OVWNNoXfxYPQ8hBln0rq_plitk5aJd7fSM&code_challenge_method=S256
```

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/4.png)

Đây là UI khi tôi tạo acc và đăng nhập thành công vào cognito. Thì khi mà thấy cognito thì tôi đoán được đây là 1 challenge liên quan đến authen, authorize. Lí do tôi đoán vậy bởi vì Cognito là một dịch vụ quản lý đăng nhập của AWS. Điều này có khả năng cách khai thác hoặc lợi dụng hệ thống xác thực để truy cập thông tin nhạy cảm hoặc backend API. Cùng confirm xem giả thuyết này đúng không nhé.

Đây là access_token khi tôi đăng nhập:

```
Headers = {
  "kid": "1+tan09ODOJDKEyVrqGX1gV84tK/84n4gcoaxf6Zr7M=",
  "alg": "RS256"
}

Payload = {
  "sub": "405c494c-b081-70f6-dd45-08cc9e42c50c",
  "iss": "https://cognito-idp.eu-north-1.amazonaws.com/eu-north-1_55iNnZmYp",
  "version": 2,
  "client_id": "1s3s6r0e517r5h7m00ab0t58ra",
  "origin_jti": "a17c7ecb-b3cf-4326-9480-b346d3ac8203",
  "event_id": "02f03301-0998-4765-b0a6-8b050f475cc8",
  "token_use": "access",
  "scope": "openid email",
  "auth_time": 1748293199,
  "exp": 1748296799,
  "iat": 1748293199,
  "jti": "a80748cd-d7be-44a1-9b84-54ce71df02a9",
  "username": "405c494c-b081-70f6-dd45-08cc9e42c50c"
}

Signature = "CZ1JyHWPnvYI6qiclMXOMazJVmJgIm_VkHuv6rFWVZaipA-4-f10NbwuneRg0CXTKC99bT2FZ0Mx7H6BYxfhhS3T-1aHSIin0rE-y6Ck1t3e8u94LsV0q4PmPZB8GogV0fPwPfASv8Jmph0lwExk84V5S3Jg9t4dQcDYj6WssIeAMNZL3hiDpglJRYFiEicCMRgUm62LWcRGe1kMbCCAgWfu8YfGHCqo-se0PkgQA0ge4JVjoyH_xKWhmOMV89VKkdx0J8YDzcPZMEulRD-qtHul2cjiZwN_fdCTJThdReVAT4yfbMX6zrObUxzP0OoyuvBE-hPH3UFq5oqM1FiEtA"
```

id_token:

```
Headers = {
  "kid": "wEkIc0pGBgrmIB2Sgww7I0oSk2kayPdzWroddAg3IQE=",
  "alg": "RS256"
}

Payload = {
  "at_hash": "5BNFgLIncn-Vy5sBxP0_HQ",
  "sub": "405c494c-b081-70f6-dd45-08cc9e42c50c",
  "email_verified": false,
  "iss": "https://cognito-idp.eu-north-1.amazonaws.com/eu-north-1_55iNnZmYp",
  "cognito:username": "405c494c-b081-70f6-dd45-08cc9e42c50c",
  "origin_jti": "a17c7ecb-b3cf-4326-9480-b346d3ac8203",
  "aud": "1s3s6r0e517r5h7m00ab0t58ra",
  "event_id": "02f03301-0998-4765-b0a6-8b050f475cc8",
  "token_use": "id",
  "auth_time": 1748293199,
  "exp": 1748296799,
  "iat": 1748293199,
  "jti": "c0cab08e-7b71-4a6f-a8fc-7f2ccfb44665",
  "email": "kudo2864@gmail.com"
}

Signature = "hhXQTpPZ02WFCyJT5gN5Avu8Ha_ttvBoE_7HVnqYh0H1lzCFFpkCZ_ITGksdqhhxJY-E5e07neDtlpcV55gHloU0C9Xh1cEIgvF2BuWTol4lJxKqxFxqklJgKzpsXKWUaA0Z0hhQZ6y85rRrtV04zBlvwd_XZJLpgRkqLsuyr9k4DKD0GoWq2t9oXZFFr3Q70nwq8Q67OfCsNVrDZ4pIYw2TsUAnYtucv13XFGroWfyecKJjZFGW51Os6dNK0T0eYeJ0hDtxxlufcdIvAPdw-SEmxL8grk0xuzKKHYtiiTe52KxUsBIPnDDRqEMDrDaE6Hh_piVp9F7UOoPOU_sb_Q"
```

refresh_token:

```
Headers = {
  "kid": "wEkIc0pGBgrmIB2Sgww7I0oSk2kayPdzWroddAg3IQE=",
  "alg": "RS256"
}

Payload = {
  "at_hash": "5BNFgLIncn-Vy5sBxP0_HQ",
  "sub": "405c494c-b081-70f6-dd45-08cc9e42c50c",
  "email_verified": false,
  "iss": "https://cognito-idp.eu-north-1.amazonaws.com/eu-north-1_55iNnZmYp",
  "cognito:username": "405c494c-b081-70f6-dd45-08cc9e42c50c",
  "origin_jti": "a17c7ecb-b3cf-4326-9480-b346d3ac8203",
  "aud": "1s3s6r0e517r5h7m00ab0t58ra",
  "event_id": "02f03301-0998-4765-b0a6-8b050f475cc8",
  "token_use": "id",
  "auth_time": 1748293199,
  "exp": 1748296799,
  "iat": 1748293199,
  "jti": "c0cab08e-7b71-4a6f-a8fc-7f2ccfb44665",
  "email": "kudo2864@gmail.com"
}

Signature = "hhXQTpPZ02WFCyJT5gN5Avu8Ha_ttvBoE_7HVnqYh0H1lzCFFpkCZ_ITGksdqhhxJY-E5e07neDtlpcV55gHloU0C9Xh1cEIgvF2BuWTol4lJxKqxFxqklJgKzpsXKWUaA0Z0hhQZ6y85rRrtV04zBlvwd_XZJLpgRkqLsuyr9k4DKD0GoWq2t9oXZFFr3Q70nwq8Q67OfCsNVrDZ4pIYw2TsUAnYtucv13XFGroWfyecKJjZFGW51Os6dNK0T0eYeJ0hDtxxlufcdIvAPdw-SEmxL8grk0xuzKKHYtiiTe52KxUsBIPnDDRqEMDrDaE6Hh_piVp9F7UOoPOU_sb_Q"
```

Đó là 3 token mà khi tôi đăng nhập thành công nó trả về, 

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/5.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/6.png)

Ta cũng có thể thấy được rằng `https://suhq95e8oj.execute-api.eu-north-1.amazonaws.com/broadcasts` unauthen, và thậm chí khi tôi cung cấp token mà tôi có được bằng cách đăng nhập vào cognito thì nó show ra api như trên.

Thì câu hỏi trong đầu tôi lúc này là liệu token với quyền cao hơn thì nó có show gì ra không. Tiếp tục check xem các thông tin khác mà tôi có ở trên.

```
"iss": "https://cognito-idp.eu-north-1.amazonaws.com/eu-north-1_55iNnZmYp"
```

Nhìn vào dòng này ta có thể thấy được token này được phát hành bởi Amazon Cognito, và `eu-north-1` là region, `eu-north-1_55iNnZmYp	` là User Pool ID

Như 1 thói quen thì tôi lại tiếp tục truy cập vào url này xem sao :"))

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/7.png)

Lỗi như trên, thì tôi tiếp tục ngứa tay, tôi lại recon :")). fuzz directory tiếp.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/8.png)

Lại có thêm tí thông tin.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/9.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/10.png)

Hmm, lần này tôi nhận ra được là scope của tôi chỉ đang có `"scope": "openid email"`, thì ý tưởng của tôi bây giờ sẽ là fake 1 access token có quyền scope cao hơn, mà đây là cognito và còn lộ 2 file `https://cognito-idp.eu-north-1.amazonaws.com/eu-north-1_55iNnZmYp/.well-known/openid-configuration` và `https://cognito-idp.eu-north-1.amazonaws.com/eu-north-1_55iNnZmYp/.well-known/jwks.json` càng làm tôi tin vào giả thuyết này :"))

Thì tôi lao vào tạo token mới token, chuyển `alg: "HS256"` và dùng `n (public key base64) làm secret key`, để tự ký token tùy ý.

Lúc này tôi chỉ thay đổi thuật toán và không thay đổi bất kì trường nào khác để confirm xem nó có đúng những gì nãy giờ tôi suy nghĩ không.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/11.png)

Fail :"))

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/12.png)

Tại nếu token hợp lệ là nó sẽ như này.

Còn vì sao tôi biết api này thì nó nằm ở 

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/13.png)

Tôi vẫn cứ có niềm tin vào cách này :")), và stuck 1 vòng luẩn quẩn quanh những dữ kiện này, nào là inject kid, tăng quyền scope,... bla bla :"))

Bất lực quá thì tôi lên mạng xem `https://hackernoon.com/application-authentication-aws-cognito-vs-json-web-token`, thì tôi nhận ra inject vào cognito là 1 chuyện điên rồ ::")), tôi chợt nhớ đến 1 đồ án môn mật mã học mà tôi đã làm vào năm 2 và cũng có sử dụng aws cognito. Thì sơ lược qua về aws cognito nó sẽ khác so với jwt truyền thống ở chỗ là việc sử dụng AWS Cognito an toàn hơn so với tự triển khai JWT truyền thống là vì Cognito do AWS quản lý hoàn toàn, bao gồm cả việc giữ private key để ký token. Trong khi đó, với JWT tự làm, dev phải tự quản lý key, dễ mắc lỗi như lộ private key, chọn thuật toán không an toàn (ví dụ alg: none), hoặc quên validate các trường quan trọng như aud, iss, ... . Vì vậy, ý tưởng này phá sản.

Nhận thấy token mà tôi nhận được khi đăng nhập qua giao diện web (được phân phối qua CloudFront) được tạo thông qua Authorization Code Flow theo tiêu chuẩn OAuth 2.0. Trong flow này, người dùng đăng nhập thông qua giao diện Hosted UI của Cognito và sau đó được redirect trở lại ứng dụng cùng với mã xác thực (authorization code), từ đó token mới được cấp. Vậy thì giờ tôi sử dụng trực tiếp qua cognito xem sao.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/14.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/15.png)

```
aws cognito-idp initiate-auth \
  --auth-flow USER_PASSWORD_AUTH \
  --client-id <APP_CLIENT_ID> \
  --auth-parameters USERNAME=<your_username>,PASSWORD=<your_password> \
  --region <your_region>
```

Tuy nhiên, khi tôi sử dụng AWS CLI và đẩy trực tiếp USERNAME và PASSWORD đến Cognito thông qua lệnh initiate-auth với --auth-flow USER_PASSWORD_AUTH, thì Cognito sẽ xác thực ngay tại chỗ và trả về token với quyền truy cập cao hơn (scope aws.cognito.signin.user.admin). Đây là một flow server side, nơi Cognito tin tưởng hơn vì thông tin xác thực được gửi trực tiếp và thông qua một App Client được cấu hình phù hợp. 

Token nhận được từ CLI không chỉ khác về scope mà còn có thể được dùng để gọi các API quản trị như get-user, điều mà token từ Authorization Code Flow không làm được. Tóm lại, cách đăng nhập vào Cognito (flow nào, từ đâu) sẽ ảnh hưởng trực tiếp đến quyền hạn của token.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/16.png)

Đã sử dụng token với scope cao hơn nhưng vẫn không được ?

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/17.png)

Chính thức stuck :")), nhìn lại request thì cũng k có thêm manh mối gì, tôi tiếp tục cắm đầu fuzz xem có thông tin gì thú vị không nhưng vẫn k ra :")).

Như một thói quen tôi bật f12 lên để tìm các file js để xem :")), bất lực r, giờ ngồi tìm trong các file js xem có gì nhạy cảm không.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/18.png)

Nhưng nó vẫn không có.

Nhưng khi tôi chuyển qua tab network thì =))

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/19.png)

một phát hiện mới ::">, tôi liền chuyển qua burp để xem nhưng trong các request gần đây lại k thấy file này, tôi liền search trong burp thì lại có, tuy nhiên chỉ có 1 request duy nhất. :")) chợt nhớ do thằng CloudFront nó cache :">, sầu thiệt chớ, vậy mà lúc đầu đâm đầu vào tương tác với UI quá nên mình bỏ sót thông tin này.

Có thể thấy rằng nó cung cấp cho ta toàn thông tin bổ ích :"))

```
aws cognito-identity get-id \
  --identity-pool-id eu-north-1:1e17b559-4464-4f84-a30f-f53974f45a65 \
  --logins "cognito-idp.eu-north-1.amazonaws.com/eu-north-1_55iNnZmYp=ID_TOKEN" \
  --region eu-north-1
```

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/20.png)

Ngon lành luôn, giờ set up env của aws cli.

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/21.png)

Với credentials này, thử truy cập các AWS services:

```
S3: aws s3 ls
Lambda: aws lambda list-functions
DynamoDB: aws dynamodb list-tables
API Gateway: aws apigateway get-rest-apis
```

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/22.png)

```
aws dynamodb describe-table --table-name ebs-broadcastsTable-2866166 --region eu-north-1
-> Lấy thông tin cấu trúc, cấu hình và metadata, không lấy dữ liệu.

aws dynamodb scan --table-name ebs-broadcastsTable-2866166 --region eu-north-1
-> Lấy toàn bộ dữ liệu bên trong bảng, quét tất cả các bản ghi.
```

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/23.png)

![image](src/content/posts/CTF/HTB/GLOBAL-CYBER-SKILLS-BENCHMARK-CTF-2025/CLOUD/EBS/24.png)

Vậy là đã giải xong challenge này.

# REFERENCE
[1]. https://machinelearningcoban.com/tabml_book/ch_model/decision_tree.html

[2]. https://viblo.asia/p/securing-aws-s3-uploads-using-presigned-urls-924lJmNmZPM

[3]. https://docs.aws.amazon.com/AmazonS3/latest/userguide/VirtualHosting.html

[4]. https://docs.aws.amazon.com/cli/latest/reference/#cli-aws

[5]. https://docs.aws.amazon.com/cognitoidentity/latest/APIReference/API_GetId.html#CognitoIdentity-GetId-request-IdentityPoolId
