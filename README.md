# An-EffcientNet-encoder-U-Net-Joint-Residual-Refinement-Module-with-TK-BUB-Loss
See our [published journal](https://www.sciencedirect.com/science/article/abs/pii/S1746809423000642?via%3Dihub) here.
#### (Specially thanks to @minhnhattrinh312 for our lovely cooperation)
## Introduction
Cell segmentation on the [2018 Data Science Bowl](https://www.kaggle.com/c/data-science-bowl-2018) Cell Nucleus Segmentation dataset; Brain Tumor segmentation on the [Brain Tumor LGG Segmentation](https://www.kaggle.com/datasets/mateuszbuda/lgg-mri-segmentation) dataset; Skin Lesion segmentation on [ISIC 2018](https://challenge.isic-archive.com/landing/2018/45/); MRI cardiac segmentation on the [ACDC](https://www.creatis.insa-lyon.fr/Challenge/acdc/databases.html) (3-D images).
## Our contributions
* We propose a network encoder with EfficientNet-B4. We have chosen the 342nd, 154th, 94th and 30th layers to attach to the skip connection. To be more detailed, the 342nd layer is the BatchNorm layer of the first MBConv block in the 6th MBConv block module; the 154th layer is the BatchNorm layer of the first MBConv block in the 4th MBConv block module; the 94th layer is the BatchNorm layer of the first MBConv block in the 3rd MBConv block module and finally, the 30th layer is the BatchNorm layer of the first MBConv block in the 2nd MBConv block module. These skip connections between pretrained encoder and decoder help preserve the spatial information in the encoder features. Although there are several ways to choose the origin of the skip-connections in the pretrained encoder, and our choice of layers is a new choice, we have considered choosing specific layers from the EfficientNetB4 for some reasons. Firstly, we focus on matching the height and width resolution of the feature maps in the encoder layer and the corresponding layer in the decoder to perform concatenating operations. Secondly, we notify the channel value that after deconvolutional operations, the difference between the input channel and the output channel is not so large, because in that case the convolutional operation might cause features loss. Lastly, when we selected specific layers, we focused on picking from the 7 MBConv blocks of the pretrained EfficientNetB4, to come in for the most salient features from the encoder. 
* In the second stage, feature maps are fed into a modified Residual Refinement Module as inspired from Qin \textit{et al.}, as we are the first to combine boundary refinement enhancement module for segmentation method. Notably, our new point is modifying Batch Normalization layers to Mean-Variance Normalization (MVN), as long as Batch Normalization could calculate windowed statistics and switch between accumulating or using fixed statistics, MVN simply centers and standardizes a single batch at a time. Additionally, Batch Normalization operations cost parameters, which results in network parameter increase; while MVN operations do not cost parameters. This normalization technique is also applied in the decoder for uniformity. We all maintain the core Residual Refinement Module (RRM), as the predicted coarse saliency maps $S_{coarse}$ is adjusted by studying the residuals $S_{residual}$ between the saliency maps and the ground truth: $S_{refined} = S_{coarse} + S_{residual}$
![Proposed Model](https://github.com/tswizzle141/An-EffcientNet-encoder-U-Net-Joint-Residual-Refinement-Module-with-TK-BUB-Loss/blob/main/1.jpg)
* To refine regional and boundary deficiency in coarse maps, this RRM exploits the residual encoder-decoder architecture, consisting of an input layer, an encoder, a bridge, a decoder and an output layer. Each contains 64 $3 \times 3$ filters, followed by a MVN layer and a ReLU non-linearity. Non-overlapping MaxPooling2D manipulates downsampling in the encoder and bilinear interpolation manipulates upsampling in the decoder. The output of this RRM is the final resulting saliency map of our model; before going through a Sigmoid or Softmax activation layer (depend on the segmentation dataset) to obtain predicted segmented output.
* The Baroni–Urbani–Buser Coefficient: $\text{BUB}=\frac{TP + \sqrt{TP \times TN}}{TP + FP + FN + \sqrt{TP \times TN}}$
* The Jaccard/Tanimoto coefficient: $\text{JT}=\frac{TP}{TP+FP+FN}$
* To the best of our knowledge, we are the first ones applying the Baroni–Urbani–Buser coefficient for segmentation problems; and we will perform two different versions of Baroni–Urbani–Buser loss functions to target the class-imbalanced issue. The new loss function for biomedical image segmentation we create is the BUB (Baroni–Urbani–Buser) loss function (version 1), which could be presented as: $l_1 = 1-BUB = \frac{FP+FN}{TP+FP+FN+\sqrt{TP \times TN}}$

We could also replace $TP$ by $\sqrt{TP \times TN}$ and keep the $TN$ quantity, thus gaining the second version of the BUB loss function: $l_2 = \frac{FP+FN}{TN+FP+FN+\sqrt{TP \times TN}}$

It could be easily seen that $0< l_1,l_2 <1$. Practices show us that the value of $TN$ always exceeds the value of $TP$ by certain times; then, if we replace $TP$ by $\sqrt{TP \times TN}$ in $l_1$, it means we try to increase the quantity $TP$, which makes the denominator value decreases then $l_1$ increases. On the other hand, if we replace $TN$ by $\sqrt{TP \times TN}$ in $l_2$, it means we try to decrease the quantity $TN$, which makes the denominator value increases then $l_2$ decreases.
* We propose to have some modifications in order to increase the model convergence speed; therefore we propose to import our BUB loss function into the Tversky-Kahneman probability weighting function for enhanced convergence speed. This probability weighting function is established as: $\omega(x)=\frac{x^{\gamma}}{[x^{\gamma}+(1-x)^{\gamma}]^{\frac{1}{\gamma}}}$
where $x \in [0,1]$ is the cumulative probability distribution of gains or losses in economical fields. To generate the TK-BUB loss function, we have: $L_{TK-BUB-v1} = \frac{l_1^{\gamma}}{[l_1^{\gamma}+(1-l_1)^{\gamma}]^{\frac{1}{\gamma}}}$
and: $L_{TK-BUB-v2} = \frac{l_2^{\gamma}}{[l_2^{\gamma}+(1-l_2)^{\gamma}]^{\frac{1}{\gamma}}}$

Above we have proposed two versions of the TK-BUB loss function. As regard to the parameter $\gamma$, experiments have been conducted with high values of $\gamma$ till it indicates that the overall result displays the best when $\gamma \in (1,2)$ and in addition, the best performance is confirmed with $\gamma=\frac{4}{3}$. Thus we train all experiments in case of $\gamma=\frac{4}{3}$. Carefully taking a look into both versions of the TK-BUB loss function, if the nominator value is powered by base $\gamma=\frac{4}{3}>1$, the denominator value is powered by base $\approx \gamma \times \frac{1}{\gamma} = 1$, as a result, the nominator value tends to converge faster than the denominator value. As can be seen that $l_1>l_2$, it follows that $L_{TK-BUB-v2}$ has a better convergence speed than $L_{TK-BUB-v1}$. As a consequence, we choose $L_{TK-BUB-v2}$ for our end-to-end training process. Tables in later sections will prove that our proposed $L_{TK-BUB-v2}$ gaining the best convergence speed as well as the most accurate segmentation results.
## Results
![table1](https://github.com/tswizzle141/An-EffcientNet-encoder-U-Net-Joint-Residual-Refinement-Module-with-TK-BUB-Loss/blob/main/2.jpg)
![table2](https://github.com/tswizzle141/An-EffcientNet-encoder-U-Net-Joint-Residual-Refinement-Module-with-TK-BUB-Loss/blob/main/3.jpg)
![table3](https://github.com/tswizzle141/An-EffcientNet-encoder-U-Net-Joint-Residual-Refinement-Module-with-TK-BUB-Loss/blob/main/4.jpg)
![table4](https://github.com/tswizzle141/An-EffcientNet-encoder-U-Net-Joint-Residual-Refinement-Module-with-TK-BUB-Loss/blob/main/5.jpg)
![table5](https://github.com/tswizzle141/An-EffcientNet-encoder-U-Net-Joint-Residual-Refinement-Module-with-TK-BUB-Loss/blob/main/6.jpg)
![table6](https://github.com/tswizzle141/An-EffcientNet-encoder-U-Net-Joint-Residual-Refinement-Module-with-TK-BUB-Loss/blob/main/7.jpg)
![table7](https://github.com/tswizzle141/An-EffcientNet-encoder-U-Net-Joint-Residual-Refinement-Module-with-TK-BUB-Loss/blob/main/8.jpg)
## Citation
If you find our work useful for your research, please cite at:

`@article{efficient_rrm,
    title = {An EffcientNet-encoder U-Net Joint Residual Refinement Module with Tversky–Kahneman Baroni–Urbani–Buser loss for biomedical image Segmentation},
    journal = {Biomedical Signal Processing and Control},
    volume = {83}, pages = {104631},
    year = {2023},
    issn = {1746-8094},
    doi = {https://doi.org/10.1016/j.bspc.2023.104631},
    url = {https://www.sciencedirect.com/science/article/pii/S1746809423000642},
    author = {Do-Hai-Ninh Nham and Minh-Nhat Trinh and Viet-Dung Nguyen and Van-Truong Pham and Thi-Thao Tran}
}
`
