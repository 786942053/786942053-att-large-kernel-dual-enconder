# 786942053-att-large-kernel-dual-enconder
# -*- coding: utf-8 -*-
# @Time    : 2021/7/8 8:59 上午
# @File    : UCTransNet.py
# @Software: PyCharm
import torch.nn as nn
import torch
import torch.nn.functional as F
from .CTrans import ChannelTransformer
from .vit_seg_modeling import Transformer,DecoderCup
#from . import vit_seg_configs as configs

def get_activation(activation_type):
    activation_type = activation_type.lower()
    if hasattr(nn, activation_type):
        return getattr(nn, activation_type)()
    else:
        return nn.ReLU()

def _make_nConv(in_channels, out_channels, nb_Conv, activation='ReLU'):
    layers = []
    layers.append(ConvBatchNorm(in_channels, out_channels, activation))

    for _ in range(nb_Conv - 1):
        layers.append(ConvBatchNorm(out_channels, out_channels, activation))
    return nn.Sequential(*layers)


class LK_encoder(nn.Module):
    def __init__(self, in_channels, out_channels, kernel_size=7, stride=1, padding=3, bias=False, batchnorm=False):
        self.in_channels = in_channels
        self.out_channels = out_channels
        self.kernel_size = kernel_size
        self.padding = padding
        self.stride = stride
        self.bias = bias
        self.batchnorm = batchnorm

        super(LK_encoder, self).__init__()

        self.layer_regularKernel = self.encoder_LK_encoder(self.in_channels, self.out_channels, kernel_size=3, stride=1,
                                                           padding=1, bias=self.bias, batchnorm=self.batchnorm)
        self.layer_largeKernel = self.encoder_LK_encoder(self.in_channels, self.out_channels,
                                                         kernel_size=self.kernel_size, stride=self.stride,
                                                         padding=self.padding, bias=self.bias, batchnorm=self.batchnorm)
        self.layer_oneKernel = self.encoder_LK_encoder(self.in_channels, self.out_channels, kernel_size=1, stride=1,
                                                       padding=0, bias=self.bias, batchnorm=self.batchnorm)
        self.attention = nn.Sequential(
            nn.Conv2d(out_channels * 3, out_channels, kernel_size=1),
            nn.Sigmoid()
        )

        self.layer_nonlinearity = nn.PReLU()
        # self.layer_batchnorm = nn.BatchNorm2d(num_features = self.out_channels)

    def encoder_LK_encoder(self, in_channels, out_channels, kernel_size=3, stride=1, padding=1, bias=False,
                           batchnorm=False):
        if batchnorm:
            layer = nn.Sequential(
                nn.Conv2d(in_channels, out_channels, kernel_size, stride=stride, padding=padding, bias=bias),
                nn.BatchNorm2d(out_channels))
        else:
            layer = nn.Sequential(
                nn.Conv2d(in_channels, out_channels, kernel_size, stride=stride, padding=padding, bias=bias))
        return layer

    def forward(self, inputs):
        # print(self.layer_regularKernel)
        regularKernel = self.layer_regularKernel(inputs)
        largeKernel = self.layer_largeKernel(inputs)
        oneKernel = self.layer_oneKernel(inputs)

        concatenated_features = torch.cat([regularKernel, largeKernel, oneKernel], dim=1)
        # 使用注意力机制动态地加权不同子模块的输出
        attention_weights = self.attention(concatenated_features)
        fused_output = regularKernel * attention_weights[:, 0:1] + largeKernel * attention_weights[:, 1:2] + oneKernel * attention_weights[:, 2:3]
        # 将输入也加入到融合的结果中
        outputs = fused_output + inputs
        # 应用激活函数
        outputs = self.layer_nonlinearity(outputs)
        return outputs
        #outputs = regularKernel + largeKernel + oneKernel + inputs
        #return self.layer_nonlinearity(outputs)

class ConvBatchNorm(nn.Module):
    """(convolution => [BN] => ReLU)"""

    def __init__(self, in_channels, out_channels, activation='ReLU'):
        super(ConvBatchNorm, self).__init__()
        self.conv = nn.Conv2d(in_channels, out_channels,
                              kernel_size=3, padding=1)
        self.norm = nn.BatchNorm2d(out_channels)
        self.activation = get_activation(activation)

    def forward(self, x):
        out = self.conv(x)
        out = self.norm(out)
        return self.activation(out)


class DownBlock(nn.Module):
    """Downscaling with maxpool convolution"""
    def __init__(self, in_channels, out_channels, nb_Conv, activation='ReLU'):
        super(DownBlock, self).__init__()
        self.maxpool = nn.MaxPool2d(2)
        self.nConvs = _make_nConv(in_channels, out_channels, nb_Conv, activation)

    def forward(self, x):
        out = self.maxpool(x)
        return self.nConvs(out)

class Flatten(nn.Module):
    def forward(self, x):
        return x.view(x.size(0), -1)

class CCA(nn.Module):
    """
    CCA Block
    """
    def __init__(self, F_g, F_x):
        super().__init__()
        self.mlp_x = nn.Sequential(
            Flatten(),
            nn.Linear(F_x, F_x))
        self.mlp_g = nn.Sequential(
            Flatten(),
            nn.Linear(F_g, F_x))
        self.relu = nn.ReLU(inplace=True)

    def forward(self, g, x):
        # channel-wise attention
        avg_pool_x = F.avg_pool2d( x, (x.size(2), x.size(3)), stride=(x.size(2), x.size(3)))
        channel_att_x = self.mlp_x(avg_pool_x)
        avg_pool_g = F.avg_pool2d( g, (g.size(2), g.size(3)), stride=(g.size(2), g.size(3)))
        channel_att_g = self.mlp_g(avg_pool_g)
        channel_att_sum = (channel_att_x + channel_att_g)/2.0
        scale = torch.sigmoid(channel_att_sum).unsqueeze(2).unsqueeze(3).expand_as(x)
        x_after_channel = x * scale
        out = self.relu(x_after_channel)
        return out

class UpBlock_attention(nn.Module):
    def __init__(self, in_channels, out_channels, nb_Conv, activation='ReLU'):
        super().__init__()
        self.up = nn.Upsample(scale_factor=2)
        self.coatt = CCA(F_g=in_channels//2, F_x=in_channels//2)
        self.nConvs = _make_nConv(in_channels, out_channels, nb_Conv, activation)

    def forward(self, x, skip_x):
        up = self.up(x)
        skip_x_att = self.coatt(g=up, x=skip_x)
        x = torch.cat([skip_x_att, up], dim=1)  # dim 1 is the channel dimension
        return self.nConvs(x)

class UCTransNet(nn.Module):
    def __init__(self, config, n_channels=3, n_classes=1,img_size=224,vis=False):
        super().__init__()
        self.vis = vis
        self.n_channels = n_channels
        self.n_classes = n_classes
        in_channels = config.base_channel

        self.inc = ConvBatchNorm(n_channels, in_channels)
        self.down1 = DownBlock(in_channels, in_channels*2, nb_Conv=2)
        self.ec1 = LK_encoder(in_channels * 2, in_channels * 2, kernel_size=7, stride=1, padding=3, bias=True)
        self.down2 = DownBlock(in_channels*2, in_channels*4, nb_Conv=2)
        self.ec2 = LK_encoder(in_channels * 4, in_channels * 4, kernel_size=7, stride=1, padding=3, bias=True)
        self.down3 = DownBlock(in_channels*4, in_channels*8, nb_Conv=2)
        self.ec3 = LK_encoder(in_channels * 8, in_channels * 8, kernel_size=7, stride=1, padding=3, bias=True)
        self.down4 = DownBlock(in_channels*8, in_channels*8, nb_Conv=2)
        self.ec4 = LK_encoder(in_channels * 8, in_channels * 8, kernel_size=7, stride=1, padding=3, bias=True)
        self.transformer = Transformer(config, img_size, vis)
        self.decoder = DecoderCup(config)
        self.mtc = ChannelTransformer(config, vis, img_size,
                                     channel_num=[in_channels, in_channels*2, in_channels*4, in_channels*8],
                                     patchSize=config.patch_sizes)
        self.up4 = UpBlock_attention(in_channels*16, in_channels*4, nb_Conv=2)
        self.up3 = UpBlock_attention(in_channels*8, in_channels*2, nb_Conv=2)

        self.up2 = UpBlock_attention(in_channels*4, in_channels, nb_Conv=2)

        self.up1 = UpBlock_attention(in_channels*2, in_channels, nb_Conv=2)

        self.outc = nn.Conv2d(in_channels, n_classes, kernel_size=(1,1), stride=(1,1))
        self.last_activation = nn.Sigmoid() # if using BCELoss

    def forward(self, x):
        x = x.float()
        x1 = self.inc(x)
        x2 = self.down1(x1)
        e1 = self.ec1(x2)
        x3 = self.down2(e1)
        e2 = self.ec2(x3)
        x4 = self.down3(e2)
        e3 = self.ec3(x4)
        x5 = self.down4(e3)
        e4 = self.ec4(x5)
        x, attn_weights, features = self.transformer(x)  # (B, n_patch, hidden)
        x = self.decoder(x, features)
        x6 = e4 + x
        x1,x2,x3,x4,att_weights = self.mtc(x1,e1,e2,e3)
        #x1,x2,x3,x4,att_weights = self.mtc(x1,x2,x3,x4)
        x = self.up4(x6, x4)
        x = self.up3(x, x3)
        x = self.up2(x, x2)
        x = self.up1(x, x1)
        if self.n_classes ==1:
            logits = self.last_activation(self.outc(x))
        else:
            logits = self.outc(x) # if nusing BCEWithLogitsLoss or class>1
        if self.vis: # visualize the attention maps
            return logits, att_weights
        else:
            return logits





