---
title: "Blogs"
layout: gridlay
sitemap: false
permalink: /blogs/
---

# Blogs

<button class="collapsible5">Terraform Provisioning</button>
<div class="content5">
    {% include_relative blogs_pages/terraform.md %}
</div>

<button class="collapsible1">Running Static Website Using Jekyll</button>
<div class="content1">
    {% include_relative blogs_pages/running_static_website_using_jekyll.md %}
</div>

<button class="collapsible4">BPF compile then load attach and run using Skeleton</button>
<div class="content2">
    {% include_relative blogs_pages/bpf_compile_build_run.md %}
</div>


<button class="collapsible2">
  <a href="/blogs_pdfs/GentooInstallationInKVM.pdf" download>
    Gentoo Linux Installation in KVM VM using genkernel
  </a>
</button>

<button class="collapsible3">
  <a href="/blogs_pdfs/DummyLinuxKernelModule.pdf" download>
    Creating a dummy Linux Kernel Module
  </a>
</button>

<script src="https://code.jquery.com/jquery-3.6.4.min.js"></script>
<script src="/_pages/blogs_pages/collapsible.js"></script>
<link rel="stylesheet" type="text/css" href="/_pages/blogs_pages/collapsible.css">
