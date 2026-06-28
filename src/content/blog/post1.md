---
title: "Classifying galaxy images with neural networks in PyTorch"
description: ""
pubDate: "Sep 10 2022"
heroImage: "/hubble.jpeg"
tags: ["Galaxies", "Neural-networks"]
---
<p> Here I present a summary of this study. The full document detailing this study and the code used can be found <a href="https://github.com/DHortaD/blog-posts/tree/main/galaxy-classification-PyTorch">here</a>. At this point in time, these results are just part of this blog post. However, I am planning to work on this a little more to turn it into a publication (hopefully soon!). If you plan on re-doing this analysis or using these results, please acknowledge this work!</p>

  <h2>Overview</h2>


  <p>Galaxies come in all shapes and sizes — round, smooth ellipticals, edge-on discs, barred spirals, merging pairs, and more. Ever since Hubble's famous "tuning fork" classification scheme in the 1920s–1930s, astronomers have been obsessed with sorting them into categories, because galaxy morphology is a window into the physics of how they assembled. In the old days this meant trained astronomers staring at images one by one. These days, we build CNNs to do it for us.</p>

  <p>This post documents a week-long experiment I undertook to build and train a convolutional neural network (CNN) in PyTorch to classify galaxy images into 10 morphological categories. This is a learning exercise and training guide — not a publishable result — but hopefully a useful one for anyone wanting to build their first image-classification CNN.</p>

  <h2>The data</h2>

  <p>The dataset used is <a href="https://astronn.readthedocs.io/en/latest/galaxy10.html">Galaxy10 DECaLS</a>, a nicely compiled set of ~17,700 galaxy images in three observing bands (g, r, z) from the DECaLS survey, classified into 10 categories via Galaxy Zoo volunteer votes. The 10 classes are: Disturbed, Merging, Round Smooth, In-between Round Smooth, Cigar Shaped Smooth, Barred Spiral, Unbarred Tight Spiral, Unbarred Loose Spiral, Edge-on without Bulge, and Edge-on with Bulge. Each image is 256×256 pixels.</p>

  <p>Before training, I cropped each image to 128×128 pixels (centred on the galaxy), split the data into training (70%), test (15%), and validation (15%) sets, and applied random horizontal/vertical flips and rotations during training. The augmentation is important: since galaxies genuinely appear upside-down, mirrored, and rotated in the sky, this encourages the model to learn morphological features rather than memorising pixel orientations.</p>

  <h2>The model architecture</h2>

  <p>The CNN consists of five convolutional blocks, each with a 3×3 kernel (padding=1 to preserve image dimensions), batch normalisation, a ReLU activation, and 2×2 max pooling. After the five blocks, the spatial dimensions have been progressively halved from 128×128 down to 4×4, while the number of feature channels grows from 3 → 32 → 64 → 128 → 256 → 512. An adaptive average pooling layer then flattens the output, a 40% dropout is applied to regularise, and a final fully connected linear layer maps the 512 features to 10 class predictions.</p>

  <p>Training used cross-entropy loss (with per-class weights to handle class imbalance) and the Adam optimiser, starting at a learning rate of 0.001 with early stopping (patience of 5 epochs) to avoid overfitting.</p>

  <h2>Results</h2>

  <p>In an initial 50-epoch run (terminating at epoch 19 via early stopping, ~34 minutes), the model climbed from ~34% training accuracy at epoch 1 to ~76% by the end — with training and test accuracies tracking closely, suggesting good generalisation and no overfitting. A subsequent fine-tuning run at a lower learning rate (0.0005) pushed this to ~80% overall accuracy across 22 more epochs (~40 minutes).</p>

  <p>The per-class breakdown on the held-out validation set tells a more nuanced story:</p>

  <table>
    <thead>
      <tr>
        <th>Galaxy class</th>
        <th>Accuracy (%)</th>
      </tr>
    </thead>
    <tbody>
      <tr><td>Edge-on without Bulge</td><td>94.5</td></tr>
      <tr><td>In-between Round Smooth</td><td>93.5</td></tr>
      <tr><td>Barred Spiral</td><td>93.0</td></tr>
      <tr><td>Edge-on with Bulge</td><td>91.3</td></tr>
      <tr><td>Round Smooth</td><td>91.2</td></tr>
      <tr><td>Cigar Shaped Smooth</td><td>90.7</td></tr>
      <tr><td>Merging</td><td>79.8</td></tr>
      <tr><td>Unbarred Tight Spiral</td><td>63.7</td></tr>
      <tr><td>Unbarred Loose Spiral</td><td>62.6</td></tr>
      <tr><td>Disturbed</td><td>40.8</td></tr>
    </tbody>
  </table>

  <p>The model excels at morphologically distinct classes — anything with a clear geometric signature (smooth roundness, a sharp edge-on disc, a prominent bar) is classified with over 90% accuracy. It struggles most with Disturbed galaxies (~41%) and has difficulty distinguishing Unbarred Tight from Unbarred Loose Spirals (~63%), both of which differ only in subtle small-scale features that max pooling tends to discard.</p>

<p> The model achieves ~80% overall accuracy on 10-class galaxy morphology classification after roughly an hour of training on a standard GPU — a solid result for a first custom CNN, and a useful baseline for future experimentation.</p>

  <h2>What could be improved</h2>

  <p>The clearest path to improvement is helping the model retain and learn from fine-grained spatial detail. Concretely: fine-tuning with smaller batch sizes and a lower learning rate to pick up on small-scale features; removing max pooling from the first two convolutional blocks to preserve more pixel information early on; and up-weighting the loss for hard classes like Disturbed galaxies. It would also be interesting to benchmark against pre-trained models like ResNet or AlexNet, which would almost certainly outperform this simple custom architecture — but building from scratch is far more instructive.</p>



</body>
</html>