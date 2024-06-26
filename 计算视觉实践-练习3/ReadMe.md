## 在Set5数据集上的图像超分辨率



本实验是对：Dong,et al. Learning a Deep Convolutional Network for Image Suer-Resolution(ECCV, 2014)的代码实现。



##### 算法原理

该篇论文首次将卷积神经网络应用到了单图像超分辨率重建领域，该方法首先采用双线性插值方法将图像进行上采样，将图像的行列数各变大为原来的2倍，之后仅仅设置了三个卷积层(每层输出都要确保特征图尺寸不变)，得到输出结果。

三层卷积分别为：

**第一层：kernel size:(9, 9); padding:4; stride:1**

**第二层：kernel size: (1, 1); padding:0; stride:1**

**第三层：kernel size:(5, 5); padding:2; stride:1**

1.针对数据集的预处理：首先采用双线性插值将图像降采样为原始尺寸的一半，之后再将其重采样为原始尺寸作为降质图像，原始图像作为标签图像。

2.训练时并不是同时对RGB三个通道同时训练，而是先将RGB图像转为YCrCb通道图像，仅对Y

通道进行训练。

3.损失函数采用了均方误差。

4.超分辨率重建时，要与训练过程一致，首先将待重建图像上采样为原始尺寸2倍，然后转为YCrCb通道，对Y通道进行重建，之后将重建后的Y通道与CrCb通道合并转换为RGB通道图像，得到重建结果。



##### 算法代码

###### 数据加载和处理

```python
import os
from torch.utils.data import Dataset
import torchvision.transforms as transform
import cv2 as cv
from PIL import Image

class dataLoader(Dataset):
    def __init__(self, imagePath, zoomFactor):
        super(DataLoader, self).__init__()
        self.imageList = [os.path.join(imagePath, file) for file in os.listdir(imagePath)]
        self.cropSize = 32
        # 按照论文方法，制作降质图像
        self.downTransform = transform.Compose([transform.CenterCrop(self.cropSize),
                                            transform.Resize(self.cropSize // zoomFactor),
                                            transform.Resize(self.cropSize, interpolation=Image.BICUBIC),
                                            transform.ToTensor()])
        # 制作label
        self.labelTransform = transform.Compose([transform.CenterCrop(self.cropSize),
                                                 transform.ToTensor()])
    def __getitem__(self, index):
        img = Image.open(self.imageList[index]).convert('YCbCr')
        y, _, _ = img.split()
        label = y.copy()
        downImg = self.downTransform(y)
        label = self.labelTransform(label)
        return downImg, label
    def __len__(self):
        return len(self.imageList)
```

###### 模型

```python
import torch.nn as nn
import torch.nn.functional as F

class SRCNN(nn.Module):
    def __init__(self):
        # 只有3个卷积层
        super(SRCNN, self).__init__() 
        # 输入通道数为1，因为只有一个Y通道；输出通道数为64
        self.conv1 = nn.Conv2d(1, 64, kernel_size=9, padding=4)  
        self.conv2 = nn.Conv2d(64, 32, kernel_size=1, padding=0)
        self.conv3 = nn.Conv2d(32, 1, kernel_size=5, padding=2)

    def forward(self, x):  # 执行流程
        out = F.relu(self.conv1(x))
        out = F.relu(self.conv2(out))
        out = self.conv3(out)
        return out
```

###### 训练并验证

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader
import math
from dataset import dataLoader
from model import SRCNN



device = torch.device("cuda:0" if (torch.cuda.is_available()) else "cpu")

# 固定一个随机数种子，方便复现训练结果
torch.manual_seed(0)
zoomFactor = 2
batchSize = 4
epochs = 50

# 训练自己的数据集的时候，可以将这里的两个路径修改一下
trainset = dataLoader("./train", zoomFactor = zoomFactor)
testset = dataLoader("./test", zoomFactor = zoomFactor)

# 加载数据集
trainloader = DataLoader(dataset=trainset, batch_size=batchSize, shuffle=True, num_workers=2)
testloader = DataLoader(dataset=testset, batch_size=batchSize, shuffle=False, num_workers=2)

# 实例化网络模型
model = SRCNN().to(device)
# 采用论文中的均方误差损失函数
criterion = nn.MSELoss()
# 在这里的优化器采用了Adam
# 对每一层的学习率都进行设置
# 其中conv1的学习率将使用外层的0.0001
optimizer = optim.Adam(
    [
        {"params": model.conv1.parameters()},
        {"params": model.conv2.parameters(), "lr": 0.0001},
        {"params": model.conv3.parameters(), "lr": 0.00001},
    ], lr=0.0001,
)

# 训练epochs轮
for epoch in range(epochs):



    epoch_loss = 0
    for iteration, batch in enumerate(trainloader):
        input, target = batch[0].to(device), batch[1].to(device)
        # 将上一次for循环的梯度清零
        optimizer.zero_grad()
        out = model(input)
        # 计算误差
        loss = criterion(out, target)
        # 反向传播
        loss.backward()
        optimizer.step()
        epoch_loss += loss.item()

    print(f"Epoch {epoch}. Training loss: {epoch_loss / len(trainloader)}")  # 打印每一个epoch和其损失率

    # 经历一个epoch训练以后，在测试集上测试一下效果
    # 用来记录平均峰值信噪比
    avg_psnr = 0
    # 测试的时候记得设置为不需要梯度下降
    with torch.no_grad():
        for batch in testloader:
            input, target = batch[0].to(device), batch[1].to(device)

            out = model(input)
            loss = criterion(out, target)

            psnr = 10 * math.log10(1 / loss.item())
            avg_psnr += psnr
    
    print(f"Average PSNR: {avg_psnr / len(testloader)} dB.")

    # 保存模型
    torch.save(model, f"model_{epoch}.pth")
```

##### PSNR、SSIM指标及超分结果

本实验在Set5上训练500个epoch，并在每个epoch进行测试得到以下结果：

**PSNR: 29.592414453705835** 

**SSIM: 0.9646698355674743**

![image-20240518222651031](./assets/image-20240518222651031-1716042417254-1.png)



但是效果并不理想