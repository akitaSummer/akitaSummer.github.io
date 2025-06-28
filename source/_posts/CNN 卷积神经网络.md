---
title: CNN 卷积神经网络
date: 2025-06-29 03:02:11
tags:
  - machine learning
  - python
  - CNN
categories: 学习笔记
---

# CNN 的由来

## MLP 有什么问题
MLP固然很美好，但是他的参数计算是相乘的，随着入参和隐藏层变多，会无法训练，比如我们要识别一张1000像素的手写图片，可能我们需要的参数就有 1000 * 100 * 100 * 10，随着参数增加，你需要的训练数据也变得不断地增加。

因此我们需要有更好的算法，来实现要求下降，还能适配特定领域问题。

## 图片特点

- locality 局部性：
    依赖性仅限于具有特征的小区域，举个例子，比如下图，鼻子的特征仅在蓝色正方形的像素间有关系，与红色矩形中的像素无关，
    ![locality](/image/cnn/locality.png)

- translation invariation 平移不变性
    举个例子，比如下图，这个鸟无论在左上还是在右下，他还是鸟，不影响被识别。
    ![translation invariation](/image/cnn/translation-invariation.png)

如果我们的模型能够抓住这两个特点，其他无关的不需要进行全连接，即可优化。

# 聚焦特点

## Convolution 卷积

卷积能完美的匹配 locality 和 translation invariation，我们来复习一下卷积的计算：

![Convolution_1](/image/cnn/convolution_1.png)
![Convolution_2](/image/cnn/convolution_2.png)
![Convolution_3](/image/cnn/convolution_3.png)
![Convolution_4](/image/cnn/convolution_4.png)

其中卷积核就是用于识别的特征提取。

## Pooling layer 池化

卷积提取完特征后，一般需要进行池化，通过模拟人类的视觉系统对数据进行降维，用更高层次的特征表示图像。一般有以下特点：

- 降低信息冗余
- 提升模型的尺度不变性、旋转不变性，识别稳定性
- 防止过拟合

![pooling](/image/cnn/pooling.png)


# LeNet

我们已经了解了卷积神经网络的所需组件，下面介绍介绍LeNet，它是最早发布的卷积神经网络之一，也是将以上所有的组件组合起来的的一个实现

![lenet](/image/cnn/lenet.png)

图中的第一层卷积后，仍保持了28*28，这是由于做了Padding，通过将数据向外补零，来避免向内坍缩

![padding](/image/cnn/padding.png)

当图片像素较大时，可通过调整步长来减少计算

![Stride](/image/cnn/stride.png)


# Coding
```python
import torch
import torch.nn as nn
import torch.optim as optim
import torchvision
import torchvision.transforms as transforms
from torch.utils.data import DataLoader

# Set random seed for reproducibility
torch.manual_seed(42)

# Define the LeNet architecture
class LeNet(nn.Module):
    def __init__(self):
        super(LeNet, self).__init__()
        self.conv0 = nn.Conv2d(1, 6, kernel_size=5)
        self.conv1 = nn.Conv2d(6, 16, kernel_size=5)
        self.fc0 = nn.Linear(16 * 4 * 4, 120)
        self.fc1 = nn.Linear(120, 84)
        self.fc2 = nn.Linear(84, 10)
        self.relu = nn.ReLU()
        self.maxpool = nn.MaxPool2d(kernel_size=2, stride=2)

    def forward(self, x):
        # Single-Channel 1-6: First conv block
        # dimension: 28x28x1 -> 24x24x6
        x = self.relu(self.conv0(x))
        # dimension: 24x24x6 -> 12x12x6
        x = self.maxpool(x)
        
        # Muti-Channel 6-16: Second conv block
        # dimension: 12x12x6 -> 8x8x16
        x = self.relu(self.conv1(x))
        # dimension: 8x8x16 -> 4x4x16
        x = self.maxpool(x)
        
        # MLP: Flatten and fully connected layers
        x = x.view(x.size(0), -1)
        # dimension: 16x4x4 -> 120
        x = self.relu(self.fc0(x))
        # dimension: 120 -> 84
        x = self.relu(self.fc1(x))
        # dimension: 84 -> 10
        x = self.fc2(x)
        return x

# Data loading and preprocessing
def load_data(batch_size=64):
    transform = transforms.Compose([
        transforms.ToTensor(),
        transforms.Normalize((0.1307,), (0.3081,))
    ])
    
    train_dataset = torchvision.datasets.MNIST(
        root='./data', 
        train=True,
        download=True, 
        transform=transform
    )
    
    test_dataset = torchvision.datasets.MNIST(
        root='./data', 
        train=False,
        download=True, 
        transform=transform
    )
    
    train_loader = DataLoader(
        train_dataset, 
        batch_size=batch_size,
        shuffle=True
    )
    
    test_loader = DataLoader(
        test_dataset, 
        batch_size=batch_size,
        shuffle=False
    )
    
    return train_loader, test_loader

# Training
def train(model, train_loader, criterion, optimizer, device):
    model.train()
    running_loss = 0.0
    correct = 0
    total = 0
    
    for images, labels in train_loader:
        images, labels = images.to(device), labels.to(device)
        
        # Do Not Forget to Zero Gradients
        optimizer.zero_grad()
        outputs = model(images)
        loss = criterion(outputs, labels)
        loss.backward()
        optimizer.step()
        
        running_loss += loss.item()
        _, predicted = outputs.max(1)
        total += labels.size(0)
        correct += predicted.eq(labels).sum().item()
    
    accuracy = 100. * correct / total
    return running_loss / len(train_loader), accuracy

# Evaluation
def evaluate(model, test_loader, criterion, device):
    model.eval()
    running_loss = 0.0
    correct = 0
    total = 0
    
    with torch.no_grad():
        for images, labels in test_loader:
            images, labels = images.to(device), labels.to(device)
            outputs = model(images)
            loss = criterion(outputs, labels)
            
            running_loss += loss.item()
            _, predicted = outputs.max(1)
            total += labels.size(0)
            correct += predicted.eq(labels).sum().item()
    
    accuracy = 100. * correct / total
    return running_loss / len(test_loader), accuracy

def main():
    # Set device
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    print(f'Using device: {device}')
    
    # Hyperparameters
    batch_size = 64
    learning_rate = 0.001
    num_epochs = 10
    
    # Load data
    train_loader, test_loader = load_data(batch_size)
    
    # Initialize model, loss function, and optimizer
    model = LeNet().to(device)
    criterion = nn.CrossEntropyLoss()
    optimizer = optim.Adam(model.parameters(), lr=learning_rate)
    
    # Training loop
    print('Starting training...')
    for epoch in range(num_epochs):
        train_loss, train_acc = train(model, train_loader, criterion, optimizer, device)
        test_loss, test_acc = evaluate(model, test_loader, criterion, device)
        
        print(f'Epoch [{epoch+1}/{num_epochs}]:')
        print(f'Train Loss: {train_loss:.4f}, Train Acc: {train_acc:.2f}%')
        print(f'Test Loss: {test_loss:.4f}, Test Acc: {test_acc:.2f}%')
        print('-' * 50)
    
    # Save the trained model
    torch.save(model.state_dict(), 'lenet_mnist.pth')
    print('Training completed and model saved!')

if __name__ == '__main__':
    main()
```