---
layout: post
author: Haidong Wang
title: "Chef soloとknife soloで環境構築を自動化"
date: 2013-10-31 20:18
comments: true
categories: [DevOps]
tags: [Chef Ruby インフラ DevOps]
description: 最新のインフラツールChefで環境構築を自動化する方法を紹介
keywords: Chef, Ruby, DevOps, Infrastructrue, インフラ自動化
---

<p class="info">
最近のInfrastructure as codeを考査しながら、一連なインフラ自動化ツールを紹介頂きたいです。本文は個人及び小規模組織の開発環境構築を自動化するツールChef soloを紹介します。ChefはC/S構造で大規模なサーバ群構築を自動化できますが、Chefサーバ構築は若干手間がかかりますので、個人向けのChef solo、knife solo及びGitを併用して、環境構築をコード化にしてみましょう。
</p>

## Chefとは？


<!-- more -->
