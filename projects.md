---
layout: page
title: Projects
sidebar_link: true
sidebar_sort_order: 4
excerpt: "Notable open source security projects released by Leonidas Tsaousis."
---

During my time in Reversec/WithSecure/F-Secure/MWR, I contributed the Kubernetes version of the "Leonidas" attack simulation framework, and published "Virtual//Attack" a collection of VMware attack techniques.   

You can find pointers to these notable open-source projects below:

<div class="project-grid">
  <a class="project-tile" href="https://github.com/ReversecLabs/virtual.attack" aria-label="View virtual slash slash attack on GitHub">
    <img class="project-tile__image" src="https://raw.githubusercontent.com/ReversecLabs/virtual.attack/main/project-logo.png" alt="virtual slash slash attack project logo" loading="lazy" />
    <span class="project-tile__content">
      <span class="project-tile__title">virtual//attack</span>
      <span class="project-tile__description">Attack techniques for threat emulation in VMware environments.</span>
      <span class="project-tile__stats" data-github-repository="ReversecLabs/virtual.attack" aria-label="GitHub repository statistics">
        <span><i class="fa fa-star" aria-hidden="true"></i> <span data-stat="stars">—</span> Stars</span>
        <span><i class="fa fa-code-fork" aria-hidden="true"></i> <span data-stat="forks">—</span> Forks</span>
      </span>
    </span>
  </a>

  <a class="project-tile" href="https://github.com/ReversecLabs/leonidas" aria-label="View leonidas on GitHub">
    <img class="project-tile__image" src="https://opengraph.githubassets.com/1/ReversecLabs/leonidas" alt="leonidas project preview" loading="lazy" />
    <span class="project-tile__content">
      <span class="project-tile__title">leonidas</span>
      <span class="project-tile__description">Automated attack simulation in the cloud, complete with detection use cases.</span>
      <span class="project-tile__stats" data-github-repository="ReversecLabs/leonidas" aria-label="GitHub repository statistics">
        <span><i class="fa fa-star" aria-hidden="true"></i> <span data-stat="stars">—</span> Stars</span>
        <span><i class="fa fa-code-fork" aria-hidden="true"></i> <span data-stat="forks">—</span> Forks</span>
      </span>
    </span>
  </a>
</div>

<script>
  (function () {
    var statisticGroups = document.querySelectorAll('[data-github-repository]');

    function formatCount(count) {
      return new Intl.NumberFormat('en').format(count);
    }

    statisticGroups.forEach(function (group) {
      var repository = group.getAttribute('data-github-repository');

      fetch('https://api.github.com/repos/' + repository, {
        headers: { 'Accept': 'application/vnd.github+json' }
      })
        .then(function (response) {
          if (!response.ok) {
            throw new Error('Unable to load GitHub repository statistics.');
          }

          return response.json();
        })
        .then(function (repositoryData) {
          group.querySelector('[data-stat="stars"]').textContent = formatCount(repositoryData.stargazers_count);
          group.querySelector('[data-stat="forks"]').textContent = formatCount(repositoryData.forks_count);
        })
        .catch(function () {
          group.hidden = true;
        });
    });
  }());
</script>