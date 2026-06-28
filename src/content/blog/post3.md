---
title: "When a machine-learning model knows something it shouldn't: interpretability in stellar spectroscopy"
description: "Machine learning interpretability in astrophysics"
pubDate: "Sep 12 2022"
heroImage: "/ai.jpg"
badge: "New"
tags: ["XAI","ML"]
---
<p> Here I present a summary of this study. The full document detailing this study can be found <a href="https://github.com/DHortaD/blog-posts/blob/main/interpreting-ML-models/Interpreting_ML_models_in_astronomy-final.pdf">here</a>. The code associated with these results will be posted soon!  </p>

<p>At this point in time, these results are just part of this blog post. However, I am planning to work on this a little more to turn it into a publication (hopefully soon!). If you plan on re-doing this analysis or using these results, please acknowledge this work or contact me!</p>

  <h2>Overview</h2>

  <p>Machine-learning models are everywhere in astronomy now, and for good reason: they are fast, flexible, and remarkably good at squeezing information out of large datasets. But there is a question we rarely ask them, and perhaps should more often: <em>how</em> are you doing this?</p>

  <p>This post is about an experiment I ran with my <a href="https://ui.adsabs.harvard.edu/abs/2025AJ....169..314H/abstract">Lux model</a> that turned into a worked example of one of the most important open problems in AI: distinguishing correlation from causation.</p>

  <h2>The setup</h2>

  <p>Stars give off light. When you split that light through a prism (or a spectrograph), you get a spectrum — a pattern of dark absorption lines at specific wavelengths, each corresponding to a chemical element absorbing photons in a star's outer atmosphere. These lines are the bar codes that let astronomers read the chemical composition of stars.</p>

  <p>The <a href="https://ui.adsabs.harvard.edu/abs/2025AJ....169..314H/abstract">Lux model</a> is a generative, multi-output, latent-variable model I developed for working with stellar spectra. One of its key capabilities is <em>label transfer</em>: given stellar spectra from one instrument (say, one operating in the near-infrared), it can map properties measured from a completely different instrument (say, one operating in the optical) onto those spectra, by training on the ~50,000 stars observed by both.</p>

  <p>For this study, I trained Lux on near-infrared spectra from the <a href="https://www.sdss.org/surveys/apogee/">APOGEE survey</a> (spanning 1.5–1.7 µm) and asked it to predict stellar labels measured optically by the <a href="https://galah-survey.org/">GALAH survey</a>. Those labels included temperature, surface gravity, iron abundance, alpha-element abundance — and <strong>europium</strong>.</p>

  <p>Here is the problem: <strong>europium has no known absorption lines in the near-infrared wavelength regime APOGEE observes.</strong> Physically, the model should not be able to infer it.</p>

  <p>And yet — it does. Very well, in fact, achieving an RMSE comparable to GALAH's own measurement uncertainty. So the question becomes: <em>how?</em></p>

  <h2>The XAI toolkit</h2>

  <p>This is precisely the kind of question that the field of mechanistic interpretability tries to answer for large language models: not "does the model perform well?" but "how does it achieve that performance, and is it doing so for the right reasons?"</p>

  <p>The worry is <strong>shortcut learning</strong>: a model achieving high accuracy not by learning a genuine causal mechanism, but by exploiting correlations in the training data that happen to yield good predictions — but may break down when those correlations don't hold.</p>

  <p>I applied four interpretability techniques to diagnose this.</p>

  <h2>1. Counterfactual residual analysis</h2>

  <p>The first test: find a star that is highly enriched in europium, find two "doppelganger" stars with near-identical chemical compositions but normal europium, and compare their near-infrared spectra directly.</p>

  <p>If the model is learning a real europium signal, there should be some spectral feature distinguishing the europium-rich star that the model is picking up on. If there is nothing to see, the model has no physical information to draw from.</p>

  <p>The result: <strong>no detectable spectral difference</strong>. The residuals between the europium-rich star and its doppelgangers show only noise and emission-line artefacts — nothing that tracks europium abundance.</p>

  <p class="takeaway"><em>Takeaway: the data gives the model nothing causal to latch onto.</em></p>

  <h2>2. Saliency mapping via Jacobian attribution</h2>

  <p>Next, I computed the Jacobian of the Lux model — specifically, how strongly each wavelength pixel covaries with the europium label. This is the stellar-spectroscopy equivalent of gradient attribution in neural networks: which input features matter most for a given prediction?</p>

  <p>The europium signal in the derivatives was sparse and localised to a handful of wavelength spikes. Crucially, when I cross-matched those spike locations against a database of known r-process atomic lines, the agreement was minimal. The one notable peak (near 15,241 Å) aligns with a <strong>cesium</strong> line — an s-process element, not an r-process one — and coincides with an emission artefact in the europium-rich star's spectrum.</p>

  <p class="takeaway"><em>Takeaway: the model is not attending to physically motivated europium lines. It is picking up noise and correlations.</em></p>

  <h2>3. Latent variable probing</h2>

  <p>Lux encodes information into a 21-dimensional latent space before projecting back to labels and spectra. I visualised how each latent dimension correlates with each label, then quantified this with <strong>Mutual Information Gap (MIG)</strong> scores — a measure of how cleanly each label is represented by a single latent dimension rather than entangled across many.</p>

  <p>The MIG scores told a clear story: europium had the lowest score of all five labels (0.02), meaning its representation in the latent space was highly entangled. Specifically, the latent dimensions that most encoded europium were the <em>same dimensions</em> that encoded alpha-element abundances and iron. The model was not building a distinct representation for europium — it was piggybacking on existing representations of correlated labels.</p>

  <p>A regression probe (training a linear model and an MLP from latents to labels) confirmed that europium had the weakest R² of all labels (~0.71), with essentially no difference between the linear and MLP probes — ruling out a hidden non-linearity. The model simply had less genuine information about europium.</p>

  <p class="takeaway"><em>Takeaway: the model's latent space doesn't meaningfully distinguish europium from the alpha elements it correlates with.</em></p>

  <h2>4. SHAP analysis</h2>

  <p>Finally, I computed SHAP (SHapley Additive exPlanations) values across all latent dimensions for each label. SHAP breaks down a model's prediction to quantify each feature's contribution.</p>

  <p>For temperature, a single latent (z₂) dominated. For iron, it was z₄. For europium, the SHAP scores were spread almost evenly across z₁–z₆ — the same dimensions encoding the other labels — with no single latent standing out. This is the fingerprint of a label being predicted indirectly, as a by-product of other correlations.</p>

  <p class="takeaway"><em>Takeaway: the model is predicting europium via its correlation with alpha-element abundances, not by finding hidden europium information in the spectra.</em></p>

  <h2>What is actually happening</h2>

  <p>Europium is an r-process element, meaning it is primarily produced in neutron star mergers and core-collapse supernovae. Alpha-elements are produced in core-collapse supernovae. In most stellar populations — particularly the kind of red giant branch stars in this sample — europium and alpha abundances are strongly correlated, because they share a nucleosynthetic history.</p>

  <p>The Lux model has learned this correlation. When asked to predict europium, it effectively asks: "what are the alpha-element and iron abundances of this star?" and maps from those. In the training data, that works. The RMSE looks good. But the model is not inferring europium from the spectra — it is inferring europium from the <em>other labels</em> it has learned from the same spectra.</p>

  <p>This is <strong>shortcut learning</strong> in a scientifically motivated, interpretable setting.</p>

  <h2>Why this matters beyond astronomy</h2>

  <p>The same question I asked of Lux — <em>is this prediction grounded in genuine understanding or statistical correlation?</em> — is the question mechanistic interpretability researchers ask of large language models every day.</p>

  <p>Whether a language model "knows" a fact because it has encoded robust causal structure, or because it has learned to pattern-match on co-occurrence statistics, is one of the central unsolved problems in AI safety and interpretability. The tools are different (circuits and attention heads rather than Jacobians and latent probes), but the epistemics are identical: <em>good performance on a test set does not establish that a model has learned the right thing.</em></p>

  <p>In stellar spectroscopy, a failure to generalise means incorrect element abundances when the europium–alpha correlation breaks down — as it does in chemically peculiar stars or certain dwarf galaxies. In higher-stakes AI applications, the consequences scale accordingly.</p>

  <p>The lesson is not that machine-learning models are untrustworthy. It is that <strong>accuracy is not the same as understanding</strong>, and that probing how a model achieves its predictions is not merely an academic exercise. It is the only way to know whether to trust a generalisation.</p>

  <h2>What next</h2>

  <p>It would be interesting to revisit this experiment with a larger sample of known r-process enriched stars, where the europium–alpha correlation may be decoupled, and to extend the interpretability analysis to more than one element. That is work for the future — but the toolkit demonstrated here is directly portable to that setting.</p>

  <p>If you want to dig into the details, the Lux paper is <a href="https://ui.adsabs.harvard.edu/abs/2025AJ....169..314H/abstract">here</a> and the code is on <a href="https://github.com/DHortaD">GitHub</a>. The notebook associated with this analysis is available on request.</p>



</body>
</html>