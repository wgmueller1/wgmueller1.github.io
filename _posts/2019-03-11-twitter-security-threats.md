---
layout: post
title: Using Machine Learning to Predict Critical Security Vulnerabilities from Twitter
description: "Our NAACL 2019 paper on analyzing cybersecurity threats on social media was featured in Wired"
modified: 2019-03-11
tags: [machine,learning,nlp,security,twitter,social,media,cybersecurity]
comments: true
published: true
image:
  feature:
  credit:
  creditlink:
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Contents</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->

I'm excited to share that our paper on using machine learning to analyze cybersecurity threats discussed on Twitter was recently <a href="https://www.wired.com/story/machine-learning-tweets-critical-security-flaws/" target="_blank">featured in Wired Magazine</a>! The work was also covered by <a href="https://securitytoday.com/articles/2019/03/11/new-system-uses-machine-learning-to-scan-tweets-for-security-flaws.aspx" target="_blank">Security Today</a> and <a href="https://cse.osu.edu/news/2019/03/ohio-state-researchers-use-natural-language-processing-identify-critical-security" target="_blank">Ohio State University</a>.

<p>
  <center><img src="/images/wired.png"><br>
  </center>
</p>


**Paper**: <a href="https://arxiv.org/abs/1902.10680" target="_blank">Analyzing the Perceived Severity of Cybersecurity Threats Reported on Social Media</a>

**Authors**: Shi Zong, Alan Ritter, Graham Mueller, Evan Wright

**Published**: NAACL 2019 (Conference of the North American Chapter of the Association for Computational Linguistics)

## The Problem

When a new software vulnerability is discovered, security professionals need to quickly assess its severity to prioritize patching efforts. However, vulnerabilities are often discussed on social media days or even weeks before they appear in official databases like the National Vulnerability Database (NVD). This creates a critical window where organizations could be vulnerable without realizing it.

Traditional approaches rely on manual analysis or waiting for official CVE (Common Vulnerabilities and Exposures) reports, which can take time. Meanwhile, security researchers, hackers, and developers are actively discussing these threats on Twitter, sharing insights about their severity and potential impact.

## Our Approach

We developed a machine learning system that:

1. **Monitors Twitter** for discussions about software vulnerabilities
2. **Links tweets to CVEs** in the National Vulnerability Database
3. **Analyzes user opinions** about vulnerability severity using natural language processing
4. **Predicts which vulnerabilities will be rated "high" or "critical"** before official ratings are published

### Dataset

We created a dataset of **6,000 annotated tweets** discussing software vulnerabilities, each labeled with perceived severity. This involved:

- Identifying tweets mentioning specific CVEs
- Extracting user opinions about threat severity
- Linking social media discussions to official vulnerability records

### Model Performance

Our model achieved impressive results:

- **Precision@50 of 0.86** when forecasting high severity vulnerabilities
- Successfully predicted which vulnerabilities would receive official "high" or "critical" severity ratings with **over 80% accuracy**
- Substantially outperformed baseline methods

## Key Findings

### Early Warning System

Twitter discussions can predict the majority of security flaws that appear in the National Vulnerability Database days later. This means security teams could get advance warning about critical threats before official CVE reports are published.

### Severity Prediction

By analyzing the language used in tweets - how urgently people discuss a vulnerability, what words they use, and the context of the discussion - we can accurately predict official severity ratings. This is valuable because:

- Official severity scores sometimes take days or weeks to be published
- The crowd's assessment often aligns with expert evaluations
- Early severity estimates help prioritize incident response

### Exploit Prediction

Perhaps most importantly, we found that social media discussions about severe vulnerabilities can predict real-world exploit activity. When people on Twitter are discussing a vulnerability with urgency, it's often a signal that exploits are being developed or are imminent.

## Why This Matters

### For Security Teams

- **Proactive defense**: Get early warnings about critical vulnerabilities before they're officially rated
- **Better prioritization**: Focus limited resources on the threats that matter most
- **Faster response**: Start patching efforts before exploits appear in the wild

### For Researchers

- **Novel data source**: Social media provides real-time, crowd-sourced threat intelligence
- **NLP applications**: Demonstrates the value of natural language processing for cybersecurity
- **Early warning signals**: Shows that online discussions contain predictive signals about real-world security events


## Future Directions

This research opens several exciting directions:

- **Real-time monitoring systems**: Deploying models that continuously scan social media for emerging threats
- **Multi-language support**: Extending beyond English tweets to capture global threat discussions
- **Integration with security tools**: Incorporating social media signals into existing vulnerability management platforms
- **Exploit prediction**: Further work on predicting not just severity, but actual exploit development and deployment

## Try It Yourself

The paper and additional materials are available on <a href="https://arxiv.org/abs/1902.10680" target="_blank">arXiv</a>. The work was presented at NAACL 2019, one of the premier conferences in natural language processing.

By combining machine learning, natural language processing, and cybersecurity expertise, we demonstrated that social media contains valuable signals for predicting and prioritizing security threats. As the security landscape continues to evolve, leveraging these data sources will become increasingly important for staying ahead of emerging vulnerabilities.

---

*This post summarizes our NAACL 2019 paper: "Analyzing the Perceived Severity of Cybersecurity Threats Reported on Social Media" by Shi Zong, Alan Ritter, Graham Mueller, and Evan Wright.*
