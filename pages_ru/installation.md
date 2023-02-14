---
title: Installation
permalink: installation.html
layout: default
sidebar: none
description: Как установить werf?
versions:
  - 1.2
  - 1.1
channels:
  - alpha
  - beta
  - ea
  - stable
  - rock-solid
arch:
  - amd64
  - arm64
setting:
  - shell
  - docker
  - kubernetes
backend:
  - docker
  - buildah
---
{%- asset installation.css %}
{%- asset installation.js %}
{%- asset releases.css %}

{%- assign releases = site.data.common.releases.releases %}
{%- assign groups = site.data.common.releases_history.history | map: "group" | uniq | reverse %}
{%- assign channels_sorted = site.data.common.channels_info.channels | sort: "stability" %}
{%- assign channels_sorted_reverse = site.data.common.channels_info.channels | sort: "stability" | reverse  %}

<div class="page__container page_installation">
  <div class="installation-selector-row">
    <div class="installation-selector">
      <div id="installation__version" class="installation-selector__title">Версия
        <span title="werf использует семантическое версионирование. <a href='/about/backward_compatibility.html' class='installation__release-channels--link'>Узнать подробнее</a>"></span>
      </div>
      <div class="tabs tabs_simple_condensed">
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="version" data-install-tab="1.2">1.2</a>
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="version" data-install-tab="1.1">1.1</a>
      </div>
    </div><!-- /selector -->
    <div class="installation-selector">
      <div id="installation__release-channels" class="installation-selector__title">Канал обновлений
        <span title="Все изменения в werf проходят через цепочку каналов обновлений. <a href='/about/release_channels.html' class='installation__release-channels--link'>Узнать подробнее</a>"></span>
      </div>
      <div class="tabs tabs_simple_condensed">
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="channel" data-install-tab="rock-solid">Rock-Solid</a>
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="channel" data-install-tab="stable">Stable</a>
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="channel" data-install-tab="ea">Early-Access</a>
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="channel" data-install-tab="beta">Beta</a>
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="channel" data-install-tab="alpha">Alpha</a>
      </div>
    </div><!-- /selector -->
  </div>
  <div class="installation-selector-row">
    <div class="installation-selector">
      <div class="installation-selector__title">Окружение запуска</div>
        <div class="tabs tabs_simple_condensed">
          <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="setting" data-install-tab="shell">Shell</a>
          <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="setting" data-install-tab="docker-setting">Docker</a>
          <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="setting" data-install-tab="kubernetes">Kubernetes</a>
        </div>
    </div>
    <div class="installation-selector">
      <div class="installation-selector__title">Сборочный бэкенд</div>
      <div class="tabs tabs_simple_condensed">
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="backend" data-install-tab="docker">Docker</a>
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="backend" data-install-tab="buildah">Buildah</a>
      </div>
    </div>
  </div><!-- /selector-row -->
  <div class="installation-selector-row">
    <div class="installation-selector">
      <div class="installation-selector__title">Операционная система</div>
      <div class="tabs tabs_simple_condensed">
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="os" data-install-tab="linux">Linux</a>
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="os" data-install-tab="macos">Mac OS</a>
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="os" data-install-tab="windows">Windows</a>
      </div>
    </div><!-- /selector -->
    <div class="installation-selector">
      <div class="installation-selector__title">Arch</div>
      <div class="tabs tabs_simple_condensed">
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="arch" data-install-tab="amd64">Amd64</a>
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="arch" data-install-tab="arm64">Arm64</a>
      </div>
    </div><!-- /selector -->
    <div class="installation-selector">
      <div class="installation-selector__title">Метод установки</div>
      <div class="tabs tabs_simple_condensed">
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="method" data-install-tab="installer">установщик</a>
        <a href="javascript:void(0)" class="tabs__btn"
          data-install-tab-group="method" data-install-tab="manually">вручную</a>
      </div>
    </div><!-- /selector -->
  </div><!-- /selector-row -->

  <div class="installation-instruction">
    <div class="docs">
      <h2 id="установка-werf">Установка</h2>
      <div class="installation-instruction__tab-content" data-install-content-group="method" data-install-content="manually">
        <div class="installation-instruction__tab-content" data-install-content-group="os" data-install-content="linux">
          {% for version in page.versions %}
          <div class="installation-instruction__tab-content" data-install-content-group="version" data-install-content="{{ version }}">
            {% for channel in page.channels %}
            <div class="installation-instruction__tab-content" data-install-content-group="channel" data-install-content="{{ channel }}">
              {% for arch in page.arch %}
              <div class="installation-instruction__tab-content" data-install-content-group="arch" data-install-content="{{ arch }}">
                <div markdown="1">
                {% include installation/trdl_linux.md version=version channel=channel arch=arch %}
                {%- if version != 1.1 %}
                {% include installation/setup_buildah.md version=version %}
                {%- endif %}
                </div>
              </div>
              {% endfor %}
            </div>
            {% endfor %}
          </div>
          {% endfor %}
        </div><!-- /os -->
        <div class="installation-instruction__tab-content" data-install-content-group="os" data-install-content="macos">
          {% for version in page.versions %}
          <div class="installation-instruction__tab-content" data-install-content-group="version" data-install-content="{{ version }}">
            {% for channel in page.channels %}
            <div class="installation-instruction__tab-content" data-install-content-group="channel" data-install-content="{{ channel }}">
              {% for arch in page.arch %}
              <div class="installation-instruction__tab-content" data-install-content-group="arch" data-install-content="{{ arch }}">
                <div markdown="1">{% include installation/trdl_macos.md version=version channel=channel arch=arch %}</div>
              </div>
              {% endfor %}
            </div>
            {% endfor %}
          </div>
          {% endfor %}
        </div><!-- /os -->
        <div class="installation-instruction__tab-content" data-install-content-group="os" data-install-content="windows">
          {% for version in page.versions %}
          <div class="installation-instruction__tab-content" data-install-content-group="version" data-install-content="{{ version }}">
            {% for channel in page.channels %}
            <div class="installation-instruction__tab-content" data-install-content-group="channel" data-install-content="{{ channel }}">
              {% for arch in page.arch %}
              <div class="installation-instruction__tab-content" data-install-content-group="arch" data-install-content="{{ arch }}">
                <div markdown="1">{% include installation/trdl_windows.md version=version channel=channel arch=arch %}</div>
              </div>
              {% endfor %}
            </div>
            {% endfor %}
          </div>
          {% endfor %}
        </div><!-- /os -->
      </div><!-- /method -->
      <div class="installation-instruction__tab-content" data-install-content-group="method" data-install-content="installer">
        <div class="installation-instruction__tab-content" data-install-content-group="os" data-install-content="linux">
          {% for version in page.versions %}
          <div class="installation-instruction__tab-content" data-install-content-group="version" data-install-content="{{ version }}">
              {% for channel in page.channels %}
              <div class="installation-instruction__tab-content" data-install-content-group="channel" data-install-content="{{ channel }}">
                {% for arch in page.arch %}
                <div class="installation-instruction__tab-content" data-install-content-group="arch" data-install-content="{{ arch }}">
                  <div markdown="1">
                  {% include installation/installer_linux_macos.md version=version channel=channel %}
                  {%- if version != 1.1 %}
                  {% include installation/setup_buildah.md version=version %}
                  {%- endif %}
                  </div>
                </div>
                {% endfor %}
              </div>
              {% endfor %}
          </div>
          {% endfor %}
        </div>
        <div class="installation-instruction__tab-content" data-install-content-group="os" data-install-content="macos">
          {% for version in page.versions %}
          <div class="installation-instruction__tab-content" data-install-content-group="version" data-install-content="{{ version }}">
              {% for channel in page.channels %}
              <div class="installation-instruction__tab-content" data-install-content-group="channel" data-install-content="{{ channel }}">
                {% for arch in page.arch %}
                <div class="installation-instruction__tab-content" data-install-content-group="arch" data-install-content="{{ arch }}">
                  <div markdown="1">
                    {% include installation/installer_linux_macos.md version=version channel=channel %}
                  </div>
                </div>
                {% endfor %}
              </div>
              {% endfor %}
          </div>
          {% endfor %}
        </div>
      </div>
    </div>
  </div>
</div>
