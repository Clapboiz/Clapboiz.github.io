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

thì sau khi tôi check thì các tỉ lệ như trên hình, bạn đọc và cũng có thể thấy. Và tôi thấy được rằng có điểm bất thường giữa `x, y` so với `z. là `x, y` bạn có thể thấy được là giao động từ `0-25`, còn `z` sẽ từ `0-1`.

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

## Cách 1: Ảnh 2 chiều
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

## Cách 2: Ảnh 3 chiều
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

