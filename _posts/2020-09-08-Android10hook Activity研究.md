---
layout:     post
title:      Android10 hook Activity研究
subtitle:   在插件化中，hook Activity作为最基本的技术，用来在宿主app中新增Activity
date:       2020-09-08
author:     duguma
header-img: img/article-bg.jpg
top: false
catalog: true
tags:
    - 系统架构
    - Android
    - Android10
    - framework
---
<style type="text/css">
html {
  font-family: sans-serif; /* 1 */
  -ms-text-size-adjust: 100%; /* 2 */
  -webkit-text-size-adjust: 100%; /* 2 */
}
body {
  margin: 0;
}
article,
aside,
details,
figcaption,
figure,
footer,
header,
hgroup,
main,
menu,
nav,
section,
summary {
  display: block;
}
audio,
canvas,
progress,
video {
  display: inline-block; /* 1 */
  vertical-align: baseline; /* 2 */
}
audio:not([controls]) {
  display: none;
  height: 0;
}
[hidden],
template {
  display: none;
}
a {
  background-color: transparent;
}
a:active,
a:hover {
  outline: 0;
}
abbr[title] {
  border-bottom: 1px dotted;
}
b,
strong {
  font-weight: bold;
}
dfn {
  font-style: italic;
}
h1 {
  font-size: 2em;
  margin: 0.67em 0;
}
mark {
  background: #ff0;
  color: #000;
}
small {
  font-size: 80%;
}
sub,
sup {
  font-size: 75%;
  line-height: 0;
  position: relative;
  vertical-align: baseline;
}
sup {
  top: -0.5em;
}
sub {
  bottom: -0.25em;
}
img {
  border: 0;
}
svg:not(:root) {
  overflow: hidden;
}
figure {
  margin: 1em 40px;
}
hr {
  -moz-box-sizing: content-box;
  box-sizing: content-box;
  height: 0;
}
pre {
  overflow: auto;
}
code,
kbd,
pre,
samp {
  font-family: monospace, monospace;
  font-size: 1em;
}
button,
input,
optgroup,
select,
textarea {
  color: inherit; /* 1 */
  font: inherit; /* 2 */
  margin: 0; /* 3 */
}
button {
  overflow: visible;
}
button,
select {
  text-transform: none;
}
button,
html input[type="button"],
input[type="reset"],
input[type="submit"] {
  -webkit-appearance: button; /* 2 */
  cursor: pointer; /* 3 */
}
button[disabled],
html input[disabled] {
  cursor: default;
}
button::-moz-focus-inner,
input::-moz-focus-inner {
  border: 0;
  padding: 0;
}
input {
  line-height: normal;
}
input[type="checkbox"],
input[type="radio"] {
  box-sizing: border-box; /* 1 */
  padding: 0; /* 2 */
}
input[type="number"]::-webkit-inner-spin-button,
input[type="number"]::-webkit-outer-spin-button {
  height: auto;
}
input[type="search"] {
  -webkit-appearance: textfield; /* 1 */
  -moz-box-sizing: content-box;
  -webkit-box-sizing: content-box; /* 2 */
  box-sizing: content-box;
}
input[type="search"]::-webkit-search-cancel-button,
input[type="search"]::-webkit-search-decoration {
  -webkit-appearance: none;
}
fieldset {
  border: 1px solid #c0c0c0;
  margin: 0 2px;
  padding: 0.35em 0.625em 0.75em;
}
legend {
  border: 0; /* 1 */
  padding: 0; /* 2 */
}
textarea {
  overflow: auto;
}
optgroup {
  font-weight: bold;
}
table {
  border-collapse: collapse;
  border-spacing: 0;
}
td,
th {
  padding: 0;
}
::selection {
  background: #262a30;
  color: #fff;
}
body {
  position: relative;
  font-family: 'Monda', "PingFang SC", "Microsoft YaHei", sans-serif;
  font-size: 16px;
  line-height: 2;
  color: #444;
  background: #f5f7f9;
}
@media (max-width: 767px) {
  body {
    padding-right: 0 !important;
  }
}
@media (min-width: 768px) and (max-width: 991px) {
  body {
    padding-right: 0 !important;
  }
}
@media (min-width: 1600px) {
  body {
    font-size: 16px;
  }
}
h1,
h2,
h3,
h4,
h5,
h6 {
  margin: 0;
  padding: 0;
  font-weight: bold;
  line-height: 1.5;
  font-family: 'Monda', "PingFang SC", "Microsoft YaHei", sans-serif;
}
h2,
h3,
h4,
h5,
h6 {
  margin: 0px 0 5px;
}
h1 {
  font-size: 22px;
}
@media (max-width: 767px) {
  h1 {
    font-size: 18px;
  }
}
h2 {
  font-size: 20px;
}
@media (max-width: 767px) {
  h2 {
    font-size: 16px;
  }
}
h3 {
  font-size: 18px;
}
@media (max-width: 767px) {
  h3 {
    font-size: 14px;
  }
}
h4 {
  font-size: 16px;
}
@media (max-width: 767px) {
  h4 {
    font-size: 12px;
  }
}
h5 {
  font-size: 14px;
}
@media (max-width: 767px) {
  h5 {
    font-size: 10px;
  }
}
h6 {
  font-size: 12px;
}
@media (max-width: 767px) {
  h6 {
    font-size: 8px;
  }
}
p {
  margin: 0 0 25px 0;
}
a {
  color: #444;
  text-decoration: none;
  border-bottom: 1px solid #999;
  word-wrap: break-word;
}
a:hover {
  color: #222;
  border-bottom-color: #222;
}
ul {
  list-style: none;
}
blockquote {
  margin: 0;
  padding: 0;
}
img {
  display: block;
  margin: auto;
  max-width: 100%;
  height: auto;
}
hr {
  margin: 40px 0;
  height: 3px;
  border: none;
  background-color: #ddd;
  background-image: repeating-linear-gradient(-45deg, #fff, #fff 4px, transparent 4px, transparent 8px);
}
blockquote {
  padding: 0 15px;
  color: #666;
  border-left: 4px solid #ddd;
}
blockquote cite::before {
  content: "-";
  padding: 0 5px;
}
dt {
  font-weight: 700;
}
dd {
  margin: 0;
  padding: 0;
}
.text-left {
  text-align: left;
}
.text-center {
  text-align: center;
}
.text-right {
  text-align: right;
}
.text-justify {
  text-align: justify;
}
.text-nowrap {
  white-space: nowrap;
}
.text-lowercase {
  text-transform: lowercase;
}
.text-uppercase {
  text-transform: uppercase;
}
.text-capitalize {
  text-transform: capitalize;
}
.center-block {
  display: block;
  margin-left: auto;
  margin-right: auto;
}
.clearfix:before,
.clearfix:after {
  content: " ";
  display: table;
}
.clearfix:after {
  clear: both;
}
.pullquote {
  width: 45%;
}
.pullquote.left {
  float: left;
  margin-left: 5px;
  margin-right: 10px;
}
.pullquote.right {
  float: right;
  margin-left: 10px;
  margin-right: 5px;
}
.affix.affix.affix {
  position: fixed;
}
.translation {
  margin-top: -20px;
  font-size: 14px;
  color: #999;
}
.scrollbar-measure {
  width: 100px;
  height: 100px;
  overflow: scroll;
  position: absolute;
  top: -9999px;
}
.use-motion .motion-element {
  opacity: 0;
}
#local-search-input {
  padding: 5px;
  border: none;
  font-size: 1.5em;
  font-weight: 800;
  line-height: 1.5em;
  text-indent: 1.5em;
  border-radius: 0;
  width: 140px;
  outline: none;
  border-bottom: 1px solid #999;
  background: inherit;
  opacity: 0.5;
  transition-duration: 200ms;
}
#local-search-input:focus {
  opacity: 1;
}
#local-search-input:focus + .search-icon {
  opacity: 1;
}
.search-icon {
  position: absolute;
  top: 16px;
  left: 15px;
  font-size: 1.5em;
  font-weight: 800;
  color: #333;
  opacity: 0.5;
  transition-duration: 200ms;
}
table {
  margin: 20px 0;
  width: 100%;
  border-collapse: collapse;
  border-spacing: 0;
  border: 1px solid #ddd;
  font-size: 14px;
  table-layout: fixed;
  word-wrap: break-all;
}
table>tbody>tr:nth-of-type(odd) {
  background-color: #f9f9f9;
}
table>tbody>tr:hover {
  background-color: #f5f5f5;
}
caption,
th,
td {
  padding: 8px;
  text-align: left;
  vertical-align: middle;
  font-weight: normal;
}
th,
td {
  border-bottom: 3px solid #ddd;
  border-right: 1px solid #eee;
}
th {
  padding-bottom: 10px;
  font-weight: 700;
}
td {
  border-bottom-width: 1px;
}
html,
body {
  height: 100%;
}
.container {
  position: relative;
  min-height: 100%;
}
.header-inner {
  margin: 0 auto;
  padding: 100px 0 70px;
  width: 800px;
}
.main {
  padding-bottom: 150px;
}
.main-inner {
  margin: 0 auto;
  width: 800px;
}
@media (min-width: 1600px) {
  .container .main-inner {
    width: 1000px;
  }
}
.footer {
  position: absolute;
  left: 0;
  bottom: 0;
  width: 100%;
  min-height: 50px;
}
.footer-inner {
  box-sizing: border-box;
  margin: 20px auto;
  width: 800px;
}
@media (min-width: 1600px) {
  .container .footer-inner {
    width: 1000px;
  }
}
pre,
.highlight {
  overflow: auto;
  margin: 20px 0;
  padding: 15px;
  font-size: 13px;
  color: #4d4d4c;
  background: #f7f7f7;
  line-height: 1.6;
}
pre,
code {
  font-family: 'consolas', consolas, Menlo, "PingFang SC", "Microsoft YaHei", monospace;
}
code {
  padding: 2px 4px;
  word-break: break-all;
  color: #444;
  background: #eee;
  border-radius: 4px;
  font-size: 13px;
}
pre code {
  padding: 0;
  color: #4d4d4c;
  background: none;
  text-shadow: none;
}
.highlight pre {
  border: none;
  margin: 0;
  padding: 1px;
}
.highlight table {
  margin: 0;
  width: auto;
  border: none;
}
.highlight td {
  border: none;
  padding: 0;
}
.highlight figcaption {
  font-size: 1em;
  color: #4d4d4c;
  line-height: 1em;
  margin-bottom: 1em;
}
.highlight figcaption:before,
.highlight figcaption:after {
  content: " ";
  display: table;
}
.highlight figcaption:after {
  clear: both;
}
.highlight figcaption a {
  float: right;
  color: #4d4d4c;
}
.highlight figcaption a:hover {
  border-bottom-color: #4d4d4c;
}
.highlight .gutter pre {
  color: #666;
  text-align: right;
  padding-right: 20px;
}
.highlight .line {
  height: 20px;
}
.gist table {
  width: auto;
}
.gist table td {
  border: none;
}
pre .comment {
  color: #8e908c;
}
pre .variable,
pre .attribute,
pre .tag,
pre .regexp,
pre .ruby .constant,
pre .xml .tag .title,
pre .xml .pi,
pre .xml .doctype,
pre .html .doctype,
pre .css .id,
pre .css .class,
pre .css .pseudo {
  color: #c82829;
}
pre .number,
pre .preprocessor,
pre .built_in,
pre .literal,
pre .params,
pre .constant,
pre .command {
  color: #f5871f;
}
pre .ruby .class .title,
pre .css .rules .attribute,
pre .string,
pre .value,
pre .inheritance,
pre .header,
pre .ruby .symbol,
pre .xml .cdata,
pre .special,
pre .number,
pre .formula {
  color: #718c00;
}
pre .title,
pre .css .hexcolor {
  color: #3e999f;
}
pre .function,
pre .python .decorator,
pre .python .title,
pre .ruby .function .title,
pre .ruby .title .keyword,
pre .perl .sub,
pre .javascript .title,
pre .coffeescript .title {
  color: #4271ae;
}
pre .keyword,
pre .javascript .function {
  color: #8959a8;
}
.full-image.full-image.full-image {
  border: none;
  max-width: 100%;
  width: auto;
  margin: 20px auto;
}
@media (min-width: 992px) {
  .full-image.full-image.full-image {
    max-width: none;
    width: 118%;
    margin: 0 -9%;
  }
}
.blockquote-center,
.page-home .post-type-quote blockquote,
.page-post-detail .post-type-quote blockquote {
  position: relative;
  margin: 40px 0;
  padding: 0;
  border-left: none;
  text-align: center;
}
.blockquote-center::before,
.page-home .post-type-quote blockquote::before,
.page-post-detail .post-type-quote blockquote::before,
.blockquote-center::after,
.page-home .post-type-quote blockquote::after,
.page-post-detail .post-type-quote blockquote::after {
  position: absolute;
  content: ' ';
  display: block;
  width: 100%;
  height: 24px;
  opacity: 0.2;
  background-repeat: no-repeat;
  background-position: 0 -6px;
  background-size: 22px 22px;
}
.blockquote-center::before,
.page-home .post-type-quote blockquote::before,
.page-post-detail .post-type-quote blockquote::before {
  top: -20px;
  background-image: url("../images/quote-l.svg");
  border-top: 1px solid #ccc;
}
.blockquote-center::after,
.page-home .post-type-quote blockquote::after,
.page-post-detail .post-type-quote blockquote::after {
  bottom: -20px;
  background-image: url("../images/quote-r.svg");
  border-bottom: 1px solid #ccc;
  background-position: 100% 8px;
}
.blockquote-center p,
.page-home .post-type-quote blockquote p,
.page-post-detail .post-type-quote blockquote p,
.blockquote-center div,
.page-home .post-type-quote blockquote div,
.page-post-detail .post-type-quote blockquote div {
  text-align: center;
}
.post .post-body .group-picture img {
  box-sizing: border-box;
  padding: 0 3px;
  border: none;
}
.post .group-picture-row {
  overflow: hidden;
  margin-top: 6px;
}
.post .group-picture-row:first-child {
  margin-top: 0;
}
.post .group-picture-column {
  float: left;
}
.page-post-detail .post-body .group-picture-column {
  float: none;
  margin-top: 10px;
  width: auto !important;
}
.page-post-detail .post-body .group-picture-column img {
  margin: 0 auto;
}
.page-archive .group-picture-container {
  overflow: hidden;
}
.page-archive .group-picture-row {
  float: left;
}
.page-archive .group-picture-row:first-child {
  margin-top: 6px;
}
.page-archive .group-picture-column {
  max-width: 150px;
  max-height: 150px;
}
.btn {
  display: inline-block;
  padding: 0 20px;
  font-size: 14px;
  color: #fff;
  background: #222;
  border: 2px solid #444;
  text-decoration: none;
  border-radius: 0;
  transition-property: background-color;
  transition-duration: 0.2s;
  transition-timing-function: ease-in-out;
  transition-delay: 0s;
}
.btn:hover,
.post-more-link .btn:hover {
  border-color: #222;
  color: #fff;
  background: #222;
}
.btn-bar {
  display: block;
  width: 22px;
  height: 2px;
  background: #444;
  border-radius: 1px;
}
.btn-bar+.btn-bar {
  margin-top: 4px;
}
.pagination {
  margin: 120px 0 40px;
  text-align: center;
  border-top: 1px solid #eee;
}
.page-number-basic,
.pagination .prev,
.pagination .next,
.pagination .page-number,
.pagination .space {
  display: inline-block;
  position: relative;
  top: -1px;
  margin: 0 10px;
  padding: 0 10px;
  line-height: 30px;
}
@media (max-width: 767px) {
  .page-number-basic,
  .pagination .prev,
  .pagination .next,
  .pagination .page-number,
  .pagination .space {
    margin: 0 5px;
  }
}
.pagination .prev,
.pagination .next,
.pagination .page-number {
  border-bottom: 0;
  border-top: 1px solid #eee;
  transition-property: border-color;
  transition-duration: 0.2s;
  transition-timing-function: ease-in-out;
  transition-delay: 0s;
}
.pagination .prev:hover,
.pagination .next:hover,
.pagination .page-number:hover {
  border-top-color: #222;
}
.pagination .space {
  padding: 0;
  margin: 0;
}
.pagination .prev {
  margin-left: 0;
}
.pagination .next {
  margin-right: 0;
}
.pagination .page-number.current {
  color: #fff;
  background: #ccc;
  border-top-color: #ccc;
}
@media (max-width: 767px) {
  .pagination {
    border-top: none;
  }
  .pagination .prev,
  .pagination .next,
  .pagination .page-number {
    margin-bottom: 10px;
    border-top: 0;
    border-bottom: 1px solid #eee;
  }
  .pagination .prev:hover,
  .pagination .next:hover,
  .pagination .page-number:hover {
    border-bottom-color: #222;
  }
}
.comments {
  margin: 60px 20px 0;
}
.tag-cloud {
  text-align: center;
}
.tag-cloud a {
  display: inline-block;
  margin: 10px;
}
.back-to-top {
  box-sizing: border-box;
  position: fixed;
  bottom: -100px;
  right: 50px;
  z-index: 1050;
  padding: 0 6px;
  width: 25px;
  background: #222;
  font-size: 12px;
  opacity: 0.6;
  color: #fff;
  cursor: pointer;
  text-align: center;
  -webkit-transform: translateZ(0);
  transition-property: bottom;
  transition-duration: 0.2s;
  transition-timing-function: ease-in-out;
  transition-delay: 0s;
}
@media (max-width: 767px) {
  .back-to-top {
    display: none;
  }
}
@media (min-width: 768px) and (max-width: 991px) {
  .back-to-top {
    display: none;
  }
}
.back-to-top.back-to-top-on {
  bottom: 40px;
}
.header {
  background: #fff;
}
.header-inner {
  position: relative;
}
.headband {
  display: none;
  height: 3px;
  background: #222;
}
.site-home {
  z-index: 2;
  position: absolute;
  top: 0;
  display: block;
  color: #fff;
  font-size: 20px;
  line-height: 20px;
  padding: 20px;
  font-weight: bolder;
  border: none;
  opacity: 0;
}
@media (min-width: 768px) and (max-width: 991px) {
  .site-home {
    font-size: 50px;
    line-height: 110px;
  }
}
.site-home:after {
  content: '';
  display: block;
  border-bottom: #fff 1px solid;
  -webkit-transform: scaleX(0);
  -moz-transform: scaleX(0);
  -ms-transform: scaleX(0);
  -o-transform: scaleX(0);
  transform: scaleX(0);
  transition-duration: 200ms;
}
.site-home:hover {
  color: #fff;
}
.site-home:hover:after {
  -webkit-transform: scaleX(1);
  -moz-transform: scaleX(1);
  -ms-transform: scaleX(1);
  -o-transform: scaleX(1);
  transform: scaleX(1);
}
.site-meta {
  text-align: center;
}
@media (max-width: 767px) {
  .site-meta {
    text-align: center;
  }
}
.brand {
  position: relative;
  display: inline-block;
  padding: 0 40px;
  color: #fff;
  background: #222;
  border-bottom: none;
}
.brand:hover {
  color: #fff;
}
.logo {
  display: inline-block;
  margin-right: 5px;
  line-height: 36px;
  vertical-align: top;
}
.site-title {
  margin-top: 10px;
  font-size: 64px;
  font-weight: 300;
  color: #fff;
}
.site-title.show:before,
.site-title.show:after {
  -webkit-transform: scaleX(1);
  -moz-transform: scaleX(1);
  -ms-transform: scaleX(1);
  -o-transform: scaleX(1);
  transform: scaleX(1);
}
@media (max-width: 767px) {
  .site-title {
    display: none;
    font-size: 60px;
  }
  .site-title:before,
  .site-title:after {
    border-width: 3px;
  }
}
.site-subtitle {
  margin-top: 10px;
  font-size: 18px;
  font-weight: 300;
  color: #fff;
}
@media (max-width: 767px) {
  .site-subtitle {
    font-size: 16px;
  }
}
.use-motion .brand {
  opacity: 0;
}
.use-motion .logo,
.use-motion .site-title,
.use-motion .site-subtitle {
  opacity: 0;
  position: relative;
  top: -10px;
}
.site-nav-toggle {
  display: none;
  position: absolute;
  top: 10px;
  left: 10px;
  opacity: 0;
}
@media (max-width: 767px) {
  .site-nav-toggle {
    display: block;
    z-index: 10;
  }
}
@media (min-width: 768px) and (max-width: 991px) {
  .site-nav-toggle {
    display: none;
  }
}
.site-nav-toggle button {
  margin-top: 2px;
  padding: 9px 10px;
  background: transparent;
  border: none;
}
.site-nav {
  z-index: 1;
  position: absolute;
  top: 0;
  right: 20px;
}
@media (max-width: 767px) {
  .site-nav {
    right: 16px;
    top: 40px;
  }
}
@media (min-width: 768px) and (max-width: 991px) {
  .site-nav {
    width: 100%;
    top: 5px;
    display: block !important;
  }
}
@media (min-width: 992px) {
  .site-nav {
    width: 100%;
    display: block !important;
  }
}
.menu {
  margin-top: 20px;
  padding-left: 0;
  text-align: center;
}
.menu .menu-item {
  display: inline-block;
  margin: 0 10px;
}
@media screen and (max-width: 767px) {
  .menu .menu-item {
    margin-top: 10px;
  }
}
.menu .menu-item a {
  display: block;
  font-size: 13px;
  text-transform: capitalize;
  line-height: inherit;
  border-bottom: 1px solid transparent;
  transition-property: border-color;
  transition-duration: 0.2s;
  transition-timing-function: ease-in-out;
  transition-delay: 0s;
}
.menu .menu-item a:hover,
.menu-item-active a {
  border-bottom-color: #222;
}
@media (max-width: 767px) {
  .menu .menu-item a {
    text-align: right !important;
  }
}
.menu .menu-item .fa {
  margin-right: 5px;
}
@media (max-width: 767px) {
  .menu .menu-item .fa {
    margin: 0 0 0 5px;
    line-height: 27px;
    float: right;
  }
}
.use-motion .menu-item {
  opacity: 0;
}
.post-body {
  font-family: 'Monda', "PingFang SC", "Microsoft YaHei", sans-serif;
}
@media (max-width: 767px) {
  .post-body {
    word-break: break-word;
  }
}
.post-body .fancybox img {
  display: block !important;
  margin: 0 auto;
  cursor: pointer;
  cursor: zoom-in;
  cursor: -webkit-zoom-in;
}
.post-body .image-caption,
.post-body .figure .caption {
  margin: 10px auto 15px;
  text-align: center;
  font-size: 16px;
  color: #999;
  font-weight: bold;
  line-height: 1;
}
.post-sticky-flag {
  display: inline-block;
  font-size: 16px;
  -ms-transform: rotate(30deg);
  -webkit-transform: rotate(30deg);
  -moz-transform: rotate(30deg);
  -ms-transform: rotate(30deg);
  -o-transform: rotate(30deg);
  transform: rotate(30deg);
}
@media (max-width: 767px) {
  .post-body pre,
  .post-body .highlight {
    padding: 10px;
  }
  .post-body pre .gutter pre,
  .post-body .highlight .gutter pre {
    padding-right: 10px;
  }
}
@media (min-width: 992px) {
  .posts-expand .post-body {
    text-align: justify;
  }
}
.posts-expand .post-body h2,
.posts-expand .post-body h3,
.posts-expand .post-body h4,
.posts-expand .post-body h5,
.posts-expand .post-body h6 {
  padding-top: 10px;
}
.posts-expand .post-body h2 .header-anchor,
.posts-expand .post-body h3 .header-anchor,
.posts-expand .post-body h4 .header-anchor,
.posts-expand .post-body h5 .header-anchor,
.posts-expand .post-body h6 .header-anchor {
  float: right;
  margin-left: 10px;
  color: #ccc;
  border-bottom-style: none;
  visibility: hidden;
}
.posts-expand .post-body h2 .header-anchor:hover,
.posts-expand .post-body h3 .header-anchor:hover,
.posts-expand .post-body h4 .header-anchor:hover,
.posts-expand .post-body h5 .header-anchor:hover,
.posts-expand .post-body h6 .header-anchor:hover {
  color: inherit;
}
.posts-expand .post-body h2:hover .header-anchor,
.posts-expand .post-body h3:hover .header-anchor,
.posts-expand .post-body h4:hover .header-anchor,
.posts-expand .post-body h5:hover .header-anchor,
.posts-expand .post-body h6:hover .header-anchor {
  visibility: visible;
}
.posts-expand .post-body ul li {
  list-style: circle;
}
.posts-expand .post-body img {
  box-sizing: border-box;
  margin: auto;
  padding: 3px;
}
.posts-expand .fancybox img {
  margin: 0 auto;
}
@media (max-width: 767px) {
  .posts-collapse {
    margin: 0 20px;
  }
  .posts-collapse .post-title,
  .posts-collapse .post-meta {
    display: block;
    width: auto;
    text-align: left;
  }
}
.posts-collapse {
  position: relative;
  z-index: 1010;
  margin-left: 55px;
}
.posts-collapse::after {
  content: " ";
  position: absolute;
  top: 20px;
  left: 0;
  margin-left: -2px;
  width: 4px;
  height: 100%;
  background: #f5f5f5;
  z-index: -1;
}
@media (max-width: 767px) {
  .posts-collapse {
    margin: 0 20px;
  }
}
.posts-collapse .collection-title {
  position: relative;
  margin: 60px 0;
}
.posts-collapse .collection-title h2 {
  margin-left: 20px;
}
.posts-collapse .collection-title small {
  color: #bbb;
}
.posts-collapse .collection-title::before {
  content: " ";
  position: absolute;
  left: 0;
  top: 50%;
  margin-left: -4px;
  margin-top: -4px;
  width: 8px;
  height: 8px;
  background: #bbb;
  border-radius: 50%;
}
.posts-collapse .post {
  margin: 30px 0;
}
.posts-collapse .post-header {
  position: relative;
  transition-duration: 0.2s;
  transition-timing-function: ease-in-out;
  transition-delay: 0s;
  transition-property: border;
  border-bottom: 1px dashed #ccc;
}
.posts-collapse .post-header::before {
  content: " ";
  position: absolute;
  left: 0;
  top: 12px;
  width: 6px;
  height: 6px;
  margin-left: -4px;
  background: #bbb;
  border-radius: 50%;
  border: 1px solid #fff;
  transition-duration: 0.2s;
  transition-timing-function: ease-in-out;
  transition-delay: 0s;
  transition-property: background;
}
.posts-collapse .post-header:hover {
  border-bottom-color: #666;
}
.posts-collapse .post-header:hover::before {
  background: #222;
}
.posts-collapse .post-meta {
  position: absolute;
  font-size: 12px;
  left: 20px;
  top: 5px;
}
.posts-collapse .post-comments-count {
  display: none;
}
.posts-collapse .post-title {
  margin-left: 60px;
  font-size: 16px;
  font-weight: normal;
  line-height: inherit;
}
.posts-collapse .post-title::after {
  margin-left: 3px;
  opacity: 0.6;
}
.posts-collapse .post-title a {
  color: #666;
  border-bottom: none;
}
.page-home .post-type-quote .post-header,
.page-post-detail .post-type-quote .post-header,
.page-home .post-type-quote .post-tags,
.page-post-detail .post-type-quote .post-tags {
  display: none;
}
.posts-expand .post-title {
  font-size: 26px;
  text-align: center;
  word-break: break-word;
  font-weight: 300;
}
@media (max-width: 767px) {
  .posts-expand .post-title {
    font-size: 22px;
  }
}
.posts-expand .post-title-link {
  display: inline-block;
  position: relative;
  color: #444;
  border-bottom: none;
  line-height: 1.2;
  vertical-align: top;
}
.posts-expand .post-title-link::before {
  content: "";
  position: absolute;
  width: 100%;
  height: 2px;
  bottom: 0;
  left: 0;
  background-color: #000;
  visibility: hidden;
  -webkit-transform: scaleX(0);
  -moz-transform: scaleX(0);
  -ms-transform: scaleX(0);
  -o-transform: scaleX(0);
  transform: scaleX(0);
  transition-duration: 0.2s;
  transition-timing-function: ease-in-out;
  transition-delay: 0s;
}
.posts-expand .post-title-link:hover::before {
  visibility: visible;
  -webkit-transform: scaleX(1);
  -moz-transform: scaleX(1);
  -ms-transform: scaleX(1);
  -o-transform: scaleX(1);
  transform: scaleX(1);
}
.posts-expand .post-title-link .fa {
  font-size: 16px;
}
.posts-expand .post-meta {
  margin: 3px 0 10px 0;
  color: #999;
  font-family: 'Monda', "PingFang SC", "Microsoft YaHei", sans-serif;
  font-size: 12px;
  text-align: center;
}
.posts-expand .post-meta .post-category-list {
  display: inline-block;
  margin: 0;
  padding: 3px;
}
.posts-expand .post-meta .post-category-list-link {
  color: #999;
}
.post-meta-item-icon {
  display: none;
  margin-right: 3px;
}
@media (min-width: 768px) and (max-width: 991px) {
  .post-meta-item-icon {
    display: inline-block;
  }
}
@media (max-width: 767px) {
  .post-meta-item-icon {
    display: inline-block;
  }
}
@media (min-width: 768px) and (max-width: 991px) {
  .post-meta-item-text {
    display: none;
  }
}
@media (max-width: 767px) {
  .post-meta-item-text {
    display: none;
  }
}
@media (max-width: 767px) {
  .posts-expand .post-comments-count {
    display: none;
  }
}
.post-more-link {
  border-bottom: 1px solid #ddd;
  padding-bottom: 0.5em;
  margin-top: 1em;
}
.post-more-link .btn {
  color: #444;
  font-size: 12px;
  background: #fff;
  border-radius: 2px;
  line-height: 2;
  margin: 0 4px 8px 4px;
  padding: 0 10px;
  border-radius: 25px;
}
.post-more-link .fa-fw {
  width: 1.285714285714286em;
  text-align: left;
}
.posts-expand .post-tags {
  margin-top: 40px;
  text-align: center;
}
.posts-expand .post-tags a {
  display: inline-block;
  margin-right: 10px;
  font-size: 13px;
}
.post-nav {
  overflow: hidden;
  margin-top: 60px;
  padding: 10px;
  white-space: nowrap;
  border-top: 1px solid #eee;
}
.post-nav-item {
  display: inline-block;
  width: 50%;
  white-space: normal;
}
.post-nav-item a {
  position: relative;
  display: inline-block;
  line-height: 25px;
  font-size: 14px;
  color: #444;
  border-bottom: none;
}
.post-nav-item a:hover {
  color: #222;
  border-bottom: none;
}
.post-nav-item a:active {
  top: 2px;
}
.post-nav-item a i {
  font-size: 12px;
}
.post-nav-prev {
  text-align: right;
}
.posts-expand .post-eof {
  display: block;
  margin: 80px auto 60px;
  width: 8%;
  height: 1px;
  background: #ccc;
  text-align: center;
}
.post:last-child .post-eof.post-eof.post-eof {
  display: none;
}
.post-gallery {
  display: table;
  table-layout: fixed;
  width: 100%;
  border-collapse: separate;
}
.post-gallery-row {
  display: table-row;
}
.post-gallery .post-gallery-img {
  display: table-cell;
  text-align: center;
  vertical-align: middle;
  border: none;
}
.post-gallery .post-gallery-img img {
  max-width: 100%;
  max-height: 100%;
  border: none;
}
.fancybox-close,
.fancybox-close:hover {
  border: none;
}
.page-post-detail .header-inner {
  overflow: visible !important;
}
.page-post-detail .site-meta,
.page-post-detail article header.post-header {
  display: none;
}
.page-post-detail .header-post {
  width: 100%;
}
.page-post-detail .post-header {
  position: relative;
  top: -10px;
  width: 960px;
  margin: 80px auto 0;
  color: #fff;
  opacity: 0;
}
@media (max-width: 767px) {
  .page-post-detail .post-header {
    width: auto;
    padding: 0 1em;
  }
}
@media (min-width: 768px) and (max-width: 991px) {
  .page-post-detail .post-header {
    width: auto;
    padding: 0 1em;
  }
}
.page-post-detail .post-header .tags a {
  display: inline-block;
  border: 1px solid rgba(255,255,255,0.8);
  border-radius: 999em;
  padding: 0 10px;
  line-height: 24px;
  color: inherit;
  font-size: 12px;
  margin: 0 1px;
  margin-bottom: 6px;
  transition-duration: 300ms;
  transition-delay: 50ms;
}
.page-post-detail .post-header .tags a:hover {
  border-color: #fff;
  background-color: rgba(255,255,255,0.4);
}
.page-post-detail .post-header h1 {
  font-size: 49px;
}
.page-post-detail .post-header time,
.page-post-detail .post-header .post-meta-item-text {
  font-family: Lora, 'Times New Roman', serif;
  font-style: italic;
  font-weight: 300;
  font-size: 20px;
}
.sidebar {
  position: fixed;
  right: 0;
  top: 0;
  bottom: 0;
  width: 0;
  z-index: 1040;
  box-shadow: inset 0 2px 6px #000;
  background: #222;
  -webkit-transform: translateZ(0);
}
.sidebar a {
  color: #999;
  border-bottom-color: #444;
}
.sidebar a:hover {
  color: #eee;
}
@media (min-width: 768px) and (max-width: 991px) {
  .sidebar {
    display: none !important;
  }
}
@media (max-width: 767px) {
  .sidebar {
    display: none !important;
  }
}
.sidebar-inner {
  position: relative;
  padding: 20px 10px;
  color: #999;
  text-align: center;
}
.sidebar-toggle {
  position: fixed;
  right: 50px;
  bottom: 45px;
  width: 15px;
  height: 15px;
  padding: 5px;
  background: #222;
  line-height: 0;
  z-index: 1050;
  cursor: pointer;
  -webkit-transform: translateZ(0);
}
@media (min-width: 768px) and (max-width: 991px) {
  .sidebar-toggle {
    display: none;
  }
}
@media (max-width: 767px) {
  .sidebar-toggle {
    display: none;
  }
}
.sidebar-toggle-line {
  position: relative;
  display: inline-block;
  vertical-align: top;
  height: 2px;
  width: 100%;
  background: #fff;
  margin-top: 3px;
}
.sidebar-toggle-line:first-child {
  margin-top: 0;
}
.site-author-image {
  display: block;
  margin: 0 auto;
  padding: 2px;
  max-width: 120px;
  height: auto;
  border: 1px solid #eee;
}
.site-author-name {
  margin: 0;
  text-align: center;
  color: #222;
  font-weight: 600;
}
.site-description {
  margin-top: 0;
  text-align: center;
  font-size: 13px;
  color: #999;
}
.site-state {
  overflow: hidden;
  line-height: 1.4;
  white-space: nowrap;
  text-align: center;
}
.site-state-item {
  display: inline-block;
  padding: 0 15px;
  border-left: 1px solid #eee;
}
.site-state-item:first-child {
  border-left: none;
}
.site-state-item a {
  border-bottom: none;
}
.site-state-item-count {
  display: block;
  text-align: center;
  color: inherit;
  font-weight: 600;
  font-size: 16px;
}
.site-state-item-name {
  font-size: 13px;
  color: #999;
}
.feed-link {
  margin-top: 20px;
}
.feed-link a {
  display: inline-block;
  padding: 0 15px;
  color: #fc6423;
  border: 1px solid #fc6423;
  border-radius: 4px;
}
.feed-link a i {
  color: #fc6423;
  font-size: 14px;
}
.feed-link a:hover {
  color: #fff;
  background: #fc6423;
}
.feed-link a:hover i {
  color: #fff;
}
.links-of-author {
  margin-top: 20px;
}
.links-of-author a {
  display: inline-block;
  vertical-align: middle;
  margin-right: 10px;
  margin-bottom: 10px;
  border-bottom-color: #444;
  font-size: 13px;
}
.links-of-author a:before {
  display: inline-block;
  vertical-align: middle;
  margin-right: 3px;
  content: " ";
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background: #f1c8ff;
}
.links-of-blogroll {
  font-size: 13px;
}
.links-of-blogroll-title {
  margin-top: 20px;
  font-size: 14px;
  font-weight: 600;
}
.links-of-blogroll-list {
  margin: 0;
  padding: 0;
}
.links-of-blogroll-item {
  padding: 2px 10px;
}
.sidebar-nav {
  margin: 0 0 20px;
  padding-left: 0;
}
.sidebar-nav li {
  display: inline-block;
  cursor: pointer;
  border-bottom: 1px solid transparent;
  font-size: 14px;
  color: #444;
}
.sidebar-nav li:hover {
  color: #fc6423;
}
.page-post-detail .sidebar-nav-toc {
  padding: 0 5px;
}
.page-post-detail .sidebar-nav-overview {
  margin-left: 10px;
}
.sidebar-nav .sidebar-nav-active {
  color: #fc6423;
  border-bottom-color: #fc6423;
}
.sidebar-nav .sidebar-nav-active:hover {
  color: #fc6423;
}
.sidebar-panel {
  display: none;
}
.sidebar-panel-active {
  display: block;
}
.post-toc-empty {
  font-size: 14px;
  color: #666;
}
.post-toc-wrap {
  overflow: hidden;
}
.post-toc {
  overflow: auto;
}
.post-toc ol {
  margin: 0;
  padding: 0 2px 5px 10px;
  text-align: left;
  list-style: none;
  font-size: 14px;
}
.post-toc ol > ol {
  padding-left: 0;
}
.post-toc ol a {
  transition-duration: 0.2s;
  transition-timing-function: ease-in-out;
  transition-delay: 0s;
  transition-property: all;
  color: #666;
  border-bottom-color: #ccc;
}
.post-toc ol a:hover {
  color: #000;
  border-bottom-color: #000;
}
.post-toc .nav-item {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  line-height: 1.8;
}
.post-toc .nav .nav-child {
  display: none;
}
.post-toc .nav .active > .nav-child {
  display: block;
}
.post-toc .nav .active-current > .nav-child {
  display: block;
}
.post-toc .nav .active-current > .nav-child > .nav-item {
  display: block;
}
.post-toc .nav .active > a {
  color: #fc6423;
  border-bottom-color: #fc6423;
}
.post-toc .nav .active-current > a {
  color: #fc6423;
}
.post-toc .nav .active-current > a:hover {
  color: #fc6423;
}
.footer {
  font-size: 14px;
  color: #999;
}
.footer img {
  border: none;
}
.footer-inner {
  text-align: center;
}
.with-love {
  display: inline-block;
  margin: 0 5px;
}
.powered-by,
.theme-info {
  display: inline-block;
}
.powered-by {
  margin-right: 10px;
}
.powered-by::after {
  content: "|";
  padding-left: 10px;
}
.cc-license {
  margin-top: 10px;
  text-align: center;
}
.cc-license .cc-opacity {
  opacity: 0.7;
  border-bottom: none;
}
.cc-license .cc-opacity:hover {
  opacity: 0.9;
}
.cc-license img {
  display: inline-block;
}
.theme-next #ds-thread #ds-reset {
  color: #555;
}
.theme-next #ds-thread #ds-reset .ds-replybox {
  margin-bottom: 30px;
}
.theme-next #ds-thread #ds-reset .ds-replybox .ds-avatar,
.theme-next #ds-reset .ds-avatar img {
  box-shadow: none;
}
.theme-next #ds-thread #ds-reset .ds-textarea-wrapper {
  border-color: #c7d4e1;
  background: none;
  border-top-right-radius: 3px;
  border-top-left-radius: 3px;
}
.theme-next #ds-thread #ds-reset .ds-textarea-wrapper textarea {
  height: 60px;
}
.theme-next #ds-reset .ds-rounded-top {
  border-radius: 0;
}
.theme-next #ds-thread #ds-reset .ds-post-toolbar {
  box-sizing: border-box;
  border: 1px solid #c7d4e1;
  background: #f6f8fa;
}
.theme-next #ds-thread #ds-reset .ds-post-options {
  height: 40px;
  border: none;
  background: none;
}
.theme-next #ds-thread #ds-reset .ds-toolbar-buttons {
  top: 11px;
}
.theme-next #ds-thread #ds-reset .ds-sync {
  top: 5px;
}
.theme-next #ds-thread #ds-reset .ds-post-button {
  top: 4px;
  right: 5px;
  width: 90px;
  height: 30px;
  border: 1px solid #c5ced7;
  border-radius: 3px;
  background-image: linear-gradient(#fbfbfc, #f5f7f9);
  color: #60676d;
}
.theme-next #ds-thread #ds-reset .ds-post-button:hover {
  background-position: 0 -30px;
  color: #60676d;
}
.theme-next #ds-thread #ds-reset .ds-comments-info {
  padding: 10px 0;
}
.theme-next #ds-thread #ds-reset .ds-sort {
  display: none;
}
.theme-next #ds-thread #ds-reset li.ds-tab a.ds-current {
  border: none;
  background: #f6f8fa;
  color: #60676d;
}
.theme-next #ds-thread #ds-reset li.ds-tab a.ds-current:hover {
  background-color: #e9f0f7;
  color: #60676d;
}
.theme-next #ds-thread #ds-reset li.ds-tab a {
  border-radius: 2px;
  padding: 5px;
}
.theme-next #ds-thread #ds-reset .ds-login-buttons p {
  color: #999;
  line-height: 36px;
}
.theme-next #ds-thread #ds-reset .ds-login-buttons .ds-service-list li {
  height: 28px;
}
.theme-next #ds-thread #ds-reset .ds-service-list a {
  background: none;
  padding: 5px;
  border: 1px solid;
  border-radius: 3px;
  text-align: center;
}
.theme-next #ds-thread #ds-reset .ds-service-list a:hover {
  color: #fff;
  background: #666;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-weibo {
  color: #fc9b00;
  border-color: #fc9b00;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-weibo:hover {
  background: #fc9b00;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-qq {
  color: #60a3ec;
  border-color: #60a3ec;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-qq:hover {
  background: #60a3ec;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-renren {
  color: #2e7ac4;
  border-color: #2e7ac4;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-renren:hover {
  background: #2e7ac4;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-douban {
  color: #37994c;
  border-color: #37994c;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-douban:hover {
  background: #37994c;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-kaixin {
  color: #fef20d;
  border-color: #fef20d;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-kaixin:hover {
  background: #fef20d;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-netease {
  color: #f00;
  border-color: #f00;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-netease:hover {
  background: #f00;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-sohu {
  color: #ffcb05;
  border-color: #ffcb05;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-sohu:hover {
  background: #ffcb05;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-baidu {
  color: #2831e0;
  border-color: #2831e0;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-baidu:hover {
  background: #2831e0;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-google {
  color: #166bec;
  border-color: #166bec;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-google:hover {
  background: #166bec;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-weixin {
  color: #00ce0d;
  border-color: #00ce0d;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-weixin:hover {
  background: #00ce0d;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-more-services {
  border: none;
}
.theme-next #ds-thread #ds-reset .ds-service-list .ds-more-services:hover {
  background: none;
}
.theme-next #ds-reset .duoshuo-ua-admin {
  display: inline-block;
  color: #f00;
}
.theme-next #ds-reset .duoshuo-ua-platform,
.theme-next #ds-reset .duoshuo-ua-browser {
  color: #ccc;
}
.theme-next #ds-reset .duoshuo-ua-platform .fa,
.theme-next #ds-reset .duoshuo-ua-browser .fa {
  display: inline-block;
  margin-right: 3px;
}
.theme-next #ds-reset .duoshuo-ua-separator {
  display: inline-block;
  margin-left: 5px;
}
.theme-next .this_ua {
  background-color: #ccc !important;
  border-radius: 4px;
  padding: 0 5px !important;
  margin: 1px 1px !important;
  border: 1px solid #bbb !important;
  color: #fff;
  display: inline-block !important;
}
.theme-next .this_ua.admin {
  background-color: #d9534f !important;
  border-color: #d9534f !important;
}
.theme-next .this_ua.platform.iOS,
.theme-next .this_ua.platform.Mac,
.theme-next .this_ua.platform.Windows {
  background-color: #39b3d7 !important;
  border-color: #46b8da !important;
}
.theme-next .this_ua.platform.Linux {
  background-color: #3a3a3a !important;
  border-color: #1f1f1f !important;
}
.theme-next .this_ua.platform.Android {
  background-color: #00c47d !important;
  border-color: #01b171 !important;
}
.theme-next .this_ua.browser.Mobile,
.theme-next .this_ua.browser.Chrome {
  background-color: #5cb85c !important;
  border-color: #4cae4c !important;
}
.theme-next .this_ua.browser.Firefox {
  background-color: #f0ad4e !important;
  border-color: #eea236 !important;
}
.theme-next .this_ua.browser.Maxthon,
.theme-next .this_ua.browser.IE {
  background-color: #428bca !important;
  border-color: #357ebd !important;
}
.theme-next .this_ua.browser.baidu,
.theme-next .this_ua.browser.UCBrowser,
.theme-next .this_ua.browser.Opera {
  background-color: #d9534f !important;
  border-color: #d43f3a !important;
}
.theme-next .this_ua.browser.Android,
.theme-next .this_ua.browser.QQBrowser {
  background-color: #78ace9 !important;
  border-color: #4cae4c !important;
}
.post-spread {
  margin-top: 20px;
  text-align: center;
}
.jiathis_style {
  display: inline-block;
}
.jiathis_style a {
  border: none;
}
.post-spread {
  margin-top: 20px;
  text-align: center;
}
.bdshare-slide-button-box a {
  border: none;
}
.bdsharebuttonbox {
  display: inline-block;
}
.bdsharebuttonbox a {
  border: none;
}
ul.search-result-list {
  padding-left: 0px;
  margin: 0px 5px 0px 8px;
}
p.search-result {
  border-bottom: 1px dashed #ccc;
  padding: 5px 0;
}
a.search-result-title {
  font-weight: bold;
}
a.search-result {
  border-bottom: transparent;
  display: block;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.search-keyword {
  border-bottom: 1px dashed #4088b8;
  font-weight: bold;
}
#local-search-result {
  height: 90%;
  overflow: auto;
}
.popup {
  display: none;
  position: fixed;
  top: 10%;
  left: 50%;
  width: 700px;
  height: 80%;
  margin-left: -350px;
  padding: 3px 10px 0;
  background: #fff;
  color: #333;
  z-index: 9999;
  border-radius: 5px;
}
@media (max-width: 767px) {
  .popup {
    padding: 3px;
    top: 0;
    left: 0;
    margin: 0;
    width: 100%;
    height: 100%;
    border-radius: 0px;
  }
}
.popoverlay {
  position: fixed;
  width: 100%;
  height: 100%;
  top: 0px;
  left: 0px;
  z-index: 2080;
  background-color: rgba(0,0,0,0.3);
}
#local-search-input {
  margin-bottom: 10px;
  width: calc(100% - 10px);
}
.popup-btn-close {
  position: absolute;
  top: 10px;
  right: 14px;
  color: #4ebd79;
  font-size: 14px;
  font-weight: bold;
  text-transform: uppercase;
  cursor: pointer;
  transition-duration: 200ms;
}
.popup-btn-close:hover {
  color: #9cce28;
}
#no-result {
  position: absolute;
  left: 44%;
  top: 42%;
  color: #ccc;
}
.busuanzi-count:before {
  content: " ";
  float: left;
  width: 260px;
  min-height: 25px;
}
@media (min-width: 768px) and (max-width: 991px) {
  .busuanzi-count {
    width: auto;
  }
  .busuanzi-count:before {
    display: none;
  }
}
@media (max-width: 767px) {
  .busuanzi-count {
    width: auto;
  }
  .busuanzi-count:before {
    display: none;
  }
}
.site-uv,
.site-pv,
.page-pv {
  display: inline-block;
}
.site-uv .busuanzi-value,
.site-pv .busuanzi-value,
.page-pv .busuanzi-value {
  margin: 0 5px;
}
.use-motion .post {
  opacity: 0;
}
.page-archive .archive-page-counter {
  position: relative;
  top: 3px;
  left: 20px;
}
@media (max-width: 767px) {
  .page-archive .archive-page-counter {
    top: 5px;
  }
}
.page-archive .posts-collapse .archive-move-on {
  position: absolute;
  top: 11px;
  left: 0;
  margin-left: -6px;
  width: 10px;
  height: 10px;
  opacity: 0.5;
  background: #444;
  border: 1px solid #fff;
  border-radius: 50%;
}
.category-all-page .category-all-title {
  text-align: center;
}
.category-all-page .category-all {
  margin-top: 20px;
}
.category-all-page .category-list {
  margin: 0;
  padding: 0;
  list-style: none;
}
.category-all-page .category-list-item {
  margin: 5px 10px;
}
.category-all-page .category-list-count {
  color: #bbb;
}
.category-all-page .category-list-count:before {
  display: inline;
  content: " (";
}
.category-all-page .category-list-count:after {
  display: inline;
  content: ") ";
}
.category-all-page .category-list-child {
  padding-left: 10px;
}
.page-post-detail .sidebar-toggle-line {
  background: #fc6423;
}
.page-post-detail .comments {
  overflow: hidden;
}
#header_post {
  min-height: 400px;
}
@media (min-width: 1600px) {
  #header_post {
    min-height: 600px;
  }
}
.header {
  position: relative;
  margin: 0 auto;
  width: 100%;
  background: no-repeat center center;
  background-color: #808080;
  background-attachment: scroll;
  background-size: cover;
  height: 220px;
}
@media (min-width: 1600px) {
  .header {
    height: 300px;
  }
}
@media (min-width: 768px) and (max-width: 991px) {
  .header {
    width: auto;
    height: 130px;
  }
}
@media (max-width: 767px) {
  .header {
    width: auto;
    height: 200px;
  }
}
.header-inner {
  top: 0;
  overflow: hidden;
  padding: 0;
  width: inherit;
}
@media (min-width: 768px) and (max-width: 991px) {
  .header-inner {
    position: relative;
    width: auto;
  }
}
@media (max-width: 767px) {
  .header-inner {
    position: relative;
    width: auto;
    overflow: visible;
  }
}
.main:before,
.main:after {
  content: " ";
  display: table;
}
.main:after {
  clear: both;
}
@media (min-width: 768px) and (max-width: 991px) {
  .main {
    padding-bottom: 100px;
  }
}
@media (max-width: 767px) {
  .main {
    padding-bottom: 100px;
  }
}
.container .main-inner {
  width: 1060px;
}
@media (min-width: 1600px) {
  .container .main-inner {
    width: 1260px;
  }
}
@media (min-width: 768px) and (max-width: 991px) {
  .container .main-inner {
    width: auto;
  }
}
@media (max-width: 767px) {
  .container .main-inner {
    width: auto;
  }
}
.content-wrap {
  float: right;
  box-sizing: border-box;
  padding: 40px;
  width: 800px;
  background: #fff;
  min-height: 700px;
}
@media (min-width: 1600px) {
  .content-wrap {
    width: 1000px;
  }
}
@media (min-width: 768px) and (max-width: 991px) {
  .content-wrap {
    width: 100%;
    padding: 1em;
  }
}
@media (max-width: 767px) {
  .content-wrap {
    width: 100%;
    padding: 1em;
    min-height: auto;
  }
}
.sidebar {
  position: static;
  float: left;
  margin-top: 300px;
  width: 240px;
  background: #fff;
  box-shadow: none;
}
@media (min-width: 768px) and (max-width: 991px) {
  .sidebar {
    display: none;
  }
}
@media (max-width: 767px) {
  .sidebar {
    display: none;
  }
}
.sidebar-toggle {
  display: none;
}
.footer-inner {
  width: 1060px;
}
.footer-inner:before {
  content: " ";
  float: left;
  width: 260px;
  min-height: 50px;
}
@media (min-width: 768px) and (max-width: 991px) {
  .footer-inner {
    width: auto;
  }
  .footer-inner:before {
    display: none;
  }
}
@media (max-width: 767px) {
  .footer-inner {
    width: auto;
  }
  .footer-inner:before {
    display: none;
  }
}
.sidebar-position-right .header-inner {
  right: 0;
}
.sidebar-position-right .content-wrap {
  float: left;
}
.sidebar-position-right .sidebar {
  float: right;
}
.sidebar-position-right .footer-inner:before {
  float: right;
}
.site-meta {
  color: #fff;
/* background: $black-deep; */
}
@media (min-width: 768px) and (max-width: 991px) {
  .site-meta {
    display: none;
  }
}
.brand {
  display: block;
  padding: 0;
  background: none;
}
.brand:hover {
  color: #fff;
}
.site-subtitle {
  margin: 0;
}
.site-search form {
  display: none;
}
.site-nav {
  border-top: none;
}
@media (max-width: 767px) {
  .site-nav {
    display: none;
    z-index: 9;
    overflow: visible !important;
  }
  .site-nav:before {
    content: '';
    position: absolute;
    top: -41px;
    right: -16px;
    width: calc(100% + 16px);
    height: calc(100% + 35px);
    background: #000;
    opacity: 0.5;
    pointer-events: none;
  }
}
@media (min-width: 768px) and (max-width: 991px) {
  .site-nav-on {
    display: block !important;
  }
}
.menu {
  margin-top: 5px;
}
.menu .menu-item {
  display: block;
  margin: 0;
  list-style: none;
}
.menu .menu-item a {
  position: relative;
  box-sizing: border-box;
  padding: 5px;
  text-align: left;
  line-height: inherit;
  color: #fff;
  transition-property: background-color;
  transition-duration: 0.2s;
  transition-timing-function: ease-in-out;
  transition-delay: 0s;
  border: none;
}
@media (max-width: 767px) {
  .menu .menu-item a {
    padding: 0 5px;
  }
}
@media (min-width: 768px) and (max-width: 991px) {
  .menu .menu-item a {
    padding: 0 5px;
  }
}
.menu .menu-item a:after {
  content: '';
  display: block;
  border-bottom: #fff 1px solid;
  -webkit-transform: scaleX(0);
  -moz-transform: scaleX(0);
  -ms-transform: scaleX(0);
  -o-transform: scaleX(0);
  transform: scaleX(0);
  transition-delay: 50ms;
  transition-duration: 200ms;
}
.menu .menu-item a:hover,
.menu-item-active a {
  color: #fff;
}
.menu .menu-item a:hover:after,
.menu-item-active a:after {
  -webkit-transform: scaleX(1);
  -moz-transform: scaleX(1);
  -ms-transform: scaleX(1);
  -o-transform: scaleX(1);
  transform: scaleX(1);
}
.menu .menu-item br {
  display: none;
}
.menu-item-active a:after {
  content: " ";
  position: absolute;
  top: 50%;
  margin-top: -3px;
  right: 15px;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background-color: #bbb;
}
.btn-bar {
  background-color: #fff;
}
.site-nav-toggle {
  top: 7px;
  right: 10px;
  left: auto;
}
.use-motion .sidebar .motion-element {
  opacity: 1;
}
.sidebar {
  display: none;
  right: auto;
  bottom: auto;
  -webkit-transform: none;
}
.sidebar-inner {
  box-sizing: border-box;
  width: 240px;
  color: #444;
  background: #fff;
}
.sidebar-inner.affix {
  position: fixed;
  top: 0;
}
.site-overview {
  margin: 0 2px;
  text-align: left;
}
.site-author:before,
.site-author:after {
  content: " ";
  display: table;
}
.site-author:after {
  clear: both;
}
.sidebar a {
  color: #444;
}
.sidebar a:hover {
  color: #222;
}
.links-of-author-item a:before {
  display: none;
}
.links-of-author-item a {
  border-bottom: none;
  text-decoration: underline;
}
.feed-link {
  border-top: 1px dotted #ccc;
  border-bottom: 1px dotted #ccc;
  text-align: center;
}
.feed-link a {
  display: block;
  color: #fc6423;
  border: none;
}
.feed-link a:hover {
  background: none;
  color: #e34603;
}
.feed-link a:hover i {
  color: #e34603;
}
.links-of-author:before,
.links-of-author:after {
  content: " ";
  display: table;
}
.links-of-author:after {
  clear: both;
}
.links-of-author-item {
  float: left;
  margin: 5px 0 0;
  width: 50%;
}
.links-of-author-item a {
  box-sizing: border-box;
  display: inline-block;
  margin-right: 0;
  margin-bottom: 0;
  padding: 0 5px;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}
.links-of-author-item a {
  display: block;
  text-decoration: none;
}
.links-of-author-item a:hover {
  border-radius: 4px;
  background: #eee;
}
.links-of-author-item .fa {
  margin-right: 2px;
  font-size: 16px;
}
.links-of-author-item .fa-globe {
  font-size: 15px;
}
.links-of-blogroll {
  margin-top: 20px;
  padding: 3px 0 0;
  border-top: 1px dotted #ccc;
}
.links-of-blogroll-title {
  margin-top: 0;
}
.links-of-blogroll-item {
  padding: 0;
}
.links-of-blogroll-inline:before,
.links-of-blogroll-inline:after {
  content: " ";
  display: table;
}
.links-of-blogroll-inline:after {
  clear: both;
}
.links-of-blogroll-inline .links-of-blogroll-item {
  float: left;
  margin: 5px 0 0;
  width: 50%;
}
.links-of-blogroll-inline .links-of-blogroll-item a {
  box-sizing: border-box;
  display: inline-block;
  margin-right: 0;
  margin-bottom: 0;
  padding: 0 5px;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}
@media (max-width: 767px) {
  .post-body {
    text-align: justify;
  }
}
.use-motion .sidebar .motion-element {
  text-align: center;
}
.main {
  margin-top: 20px;
}
.menu {
  overflow: hidden;
}
.menu .menu-item a {
  font-size: 12px;
  line-height: 12px;
  padding: 20px;
  float: right;
}
@media (max-width: 767px) {
  .menu .menu-item a {
    font-size: 12px;
    line-height: inherit;
    padding: 0px 5px;
  }
}
.header-inner-post {
  padding-bottom: 60px;
}
.header-inner {
  position: relative;
  height: 100%;
}
.site-meta {
  position: relative;
  height: 100%;
  width: 100px;
  margin: 0 auto;
}
.site-meta-headline {
  text-align: center;
  height: 40px;
  width: 100%;
  position: absolute;
  top: 48%;
  margin-top: -20px;
}
.posts-expand .post-title {
  text-align: left;
  font-weight: 400;
  border-bottom: 0px;
}
.posts-expand .post-meta {
  text-align: left;
  margin-top: -10px;
}
.posts-expand .post-eof {
  display: none;
  margin: 20px auto 40px;
}
h1 {
  border-bottom: 1px solid #ccc;
  padding-bottom: 6px;
  margin-top: 15px;
}
.page-post-detail .post-header h1 {
  border-bottom: 0px;
}
p {
  margin: 10px 0 10px 0;
}
body {
  line-height: 1.6;
}
pre,
.highlight {
  margin: 10px 0;
}


</style>
        
        <p>在插件化中，hook Activity作为最基本的技术，用来在宿主app中新增Activity，而通常情况下，Activity必须在Manifest中注册在才可以使用，下面将就Android10.0来分析hook Activity的详细过程。</p>
<p>要hook Activity之前，必须知道Activity的启动过程，才能够选择合适的点进行hook，在前面的文章中有分析<a href="https://skytoby.github.io/2019/startActivity%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/" target="_blank" rel="noopener">Activity详细的启动过程</a>，hook主要是两个点：一是在Activity给AMS之前替换代理的Activity，二是在handler中发送启动Activity时替换为插件的Activity。</p>
<p>分析完了Activity的hook点之后，还有一个重要的问题，如何加载插件的类和资源，只有加载了插件中的类和资源，后面的hook才有意义。</p>
<h3 id="一、类加载"><a href="#一、类加载" class="headerlink" title="一、类加载"></a>一、类加载</h3><p>在Android中，将代码编译后会生成apk文件，apk文件里面有一个或多个classes.dex文件，它是所有class文件进行合并，优化后生成。在apk运行时ART虚拟机或Dalvik虚拟机会加载dex文件，加载都是通过ClassLoader实现。</p>
<p>ClassLoader是一个抽象类，实现分为系统类加载器和自定义类加载器。</p>
<p>系统类加载器有三种：</p>
<p>BootClassLoader：用于加载Android Framework中class文件。</p>
<p>PathClassLoader：用于Android应用程序类加载器，可以加载指定的dex，以及jar、zip、apk中的dex。</p>
<p>DexClassLoader：用于加载指定的dex，以及jar、zip、apk中的dex。</p>
<h4 id="1-1-PathClassLoader"><a href="#1-1-PathClassLoader" class="headerlink" title="1.1 PathClassLoader"></a>1.1 PathClassLoader</h4><p>[-&gt;libcore\dalvik\src\main\java\dalvik\system\PathClassLoader.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br></pre></td><td class="code"><pre><span class="line">public class PathClassLoader extends BaseDexClassLoader &#123;</span><br><span class="line">    /**</span><br><span class="line">     * Creates a &#123;@code PathClassLoader&#125; that operates on a given list of files</span><br><span class="line">     * and directories. This method is equivalent to calling</span><br><span class="line">     * &#123;@link #PathClassLoader(String, String, ClassLoader)&#125; with a</span><br><span class="line">     * &#123;@code null&#125; value for the second argument (see description there).</span><br><span class="line">     *</span><br><span class="line">     * @param dexPath the list of jar/apk files containing classes and</span><br><span class="line">     * resources, delimited by &#123;@code File.pathSeparator&#125;, which</span><br><span class="line">     * defaults to &#123;@code &quot;:&quot;&#125; on Android</span><br><span class="line">     * @param parent the parent class loader</span><br><span class="line">     */</span><br><span class="line">    public PathClassLoader(String dexPath, ClassLoader parent) &#123;</span><br><span class="line">        super(dexPath, null, null, parent);</span><br><span class="line">    &#125;</span><br><span class="line"></span><br><span class="line">    /**</span><br><span class="line">     * Creates a &#123;@code PathClassLoader&#125; that operates on two given</span><br><span class="line">     * lists of files and directories. The entries of the first list</span><br><span class="line">     * should be one of the following:</span><br><span class="line">     *</span><br><span class="line">     * &lt;ul&gt;</span><br><span class="line">     * &lt;li&gt;JAR/ZIP/APK files, possibly containing a &quot;classes.dex&quot; file as</span><br><span class="line">     * well as arbitrary resources.</span><br><span class="line">     * &lt;li&gt;Raw &quot;.dex&quot; files (not inside a zip file).</span><br><span class="line">     * &lt;/ul&gt;</span><br><span class="line">     *</span><br><span class="line">     * The entries of the second list should be directories containing</span><br><span class="line">     * native library files.</span><br><span class="line">     *</span><br><span class="line">     * @param dexPath the list of jar/apk files containing classes and</span><br><span class="line">     * resources, delimited by &#123;@code File.pathSeparator&#125;, which</span><br><span class="line">     * defaults to &#123;@code &quot;:&quot;&#125; on Android</span><br><span class="line">     * @param librarySearchPath the list of directories containing native</span><br><span class="line">     * libraries, delimited by &#123;@code File.pathSeparator&#125;; may be</span><br><span class="line">     * &#123;@code null&#125;</span><br><span class="line">     * @param parent the parent class loader</span><br><span class="line">     */</span><br><span class="line">    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) &#123;</span><br><span class="line">        super(dexPath, null, librarySearchPath, parent);</span><br><span class="line">    &#125;</span><br><span class="line">&#125;</span><br></pre></td></tr></table></figure>
<p>BaseDexClassLoader</p>
<p>[-&gt;libcore\dalvik\src\main\java\dalvik\system\BaseDexClassLoader.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br></pre></td><td class="code"><pre><span class="line">public class BaseDexClassLoader extends ClassLoader &#123;</span><br><span class="line"></span><br><span class="line">    /**</span><br><span class="line">     * Hook for customizing how dex files loads are reported.</span><br><span class="line">     *</span><br><span class="line">     * This enables the framework to monitor the use of dex files. The</span><br><span class="line">     * goal is to simplify the mechanism for optimizing foreign dex files and</span><br><span class="line">     * enable further optimizations of secondary dex files.</span><br><span class="line">     *</span><br><span class="line">     * The reporting happens only when new instances of BaseDexClassLoader</span><br><span class="line">     * are constructed and will be active only after this field is set with</span><br><span class="line">     * &#123;@link BaseDexClassLoader#setReporter&#125;.</span><br><span class="line">     */</span><br><span class="line">    /* @NonNull */ private static volatile Reporter reporter = null;</span><br><span class="line"></span><br><span class="line">    private final DexPathList pathList;</span><br><span class="line">    .....</span><br><span class="line">&#125;</span><br></pre></td></tr></table></figure>
<h4 id="1-2-DexClassLoader"><a href="#1-2-DexClassLoader" class="headerlink" title="1.2 DexClassLoader"></a>1.2 DexClassLoader</h4><p>[-&gt;libcore\dalvik\src\main\java\dalvik\system\DexClassLoader.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br></pre></td><td class="code"><pre><span class="line">public class DexClassLoader extends BaseDexClassLoader &#123;</span><br><span class="line">    /**</span><br><span class="line">     * Creates a &#123;@code DexClassLoader&#125; that finds interpreted and native</span><br><span class="line">     * code.  Interpreted classes are found in a set of DEX files contained</span><br><span class="line">     * in Jar or APK files.</span><br><span class="line">     *</span><br><span class="line">     * &lt;p&gt;The path lists are separated using the character specified by the</span><br><span class="line">     * &#123;@code path.separator&#125; system property, which defaults to &#123;@code :&#125;.</span><br><span class="line">     *</span><br><span class="line">     * @param dexPath the list of jar/apk files containing classes and</span><br><span class="line">     *     resources, delimited by &#123;@code File.pathSeparator&#125;, which</span><br><span class="line">     *     defaults to &#123;@code &quot;:&quot;&#125; on Android</span><br><span class="line">     * @param optimizedDirectory this parameter is deprecated and has no effect since API level 26.</span><br><span class="line">     * @param librarySearchPath the list of directories containing native</span><br><span class="line">     *     libraries, delimited by &#123;@code File.pathSeparator&#125;; may be</span><br><span class="line">     *     &#123;@code null&#125;</span><br><span class="line">     * @param parent the parent class loader</span><br><span class="line">     */</span><br><span class="line">    public DexClassLoader(String dexPath, String optimizedDirectory,</span><br><span class="line">            String librarySearchPath, ClassLoader parent) &#123;</span><br><span class="line">        super(dexPath, null, librarySearchPath, parent);</span><br><span class="line">    &#125;</span><br><span class="line">&#125;</span><br></pre></td></tr></table></figure>
<p>PathClassLoader和DexClassLoader两者都是继承于BaseDexClassLoader，并且类中只有构成方法，实现全部在BaseDexClassLoader中。从源码中可以看出DexClassLoader多个一个optimizedDirectory参数，但是实际上没什么用处，这两者最后调用的super方法一模一样。</p>
<p><img src="/2020/Android10.0如何hook Activity/classloader.PNG" alt="classloader" style="zoom: 67%;"></p>
<h4 id="1-3-加载原理"><a href="#1-3-加载原理" class="headerlink" title="1.3 加载原理"></a>1.3 加载原理</h4><p>类加载器通过loadClass方法加载apk文件中的类。</p>
<h5 id="1-3-1-ClassLoader-loadClass"><a href="#1-3-1-ClassLoader-loadClass" class="headerlink" title="1.3.1 ClassLoader.loadClass"></a>1.3.1 ClassLoader.loadClass</h5><p>[-&gt;libcore\ojluni\src\main\java\java\lang\ClassLoader.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br><span class="line">43</span><br></pre></td><td class="code"><pre><span class="line">protected Class&lt;?&gt; loadClass(String name, boolean resolve)</span><br><span class="line">        throws ClassNotFoundException</span><br><span class="line">    &#123;</span><br><span class="line">            // First, check if the class has already been loaded</span><br><span class="line">            // 检查这个类是否已经被加载</span><br><span class="line">            Class&lt;?&gt; c = findLoadedClass(name);</span><br><span class="line">            if (c == null) &#123;</span><br><span class="line">                try &#123;</span><br><span class="line">                    if (parent != null) &#123;</span><br><span class="line">                        //如果parent不为空，则调用parent的loadClass进行加载</span><br><span class="line">                        c = parent.loadClass(name, false);</span><br><span class="line">                    &#125; else &#123;</span><br><span class="line">                        //正常情况下不会走到这里</span><br><span class="line">                        c = findBootstrapClassOrNull(name);</span><br><span class="line">                    &#125;</span><br><span class="line">                &#125; catch (ClassNotFoundException e) &#123;</span><br><span class="line">                    // ClassNotFoundException thrown if class not found</span><br><span class="line">                    // from the non-null parent class loader</span><br><span class="line">                &#125;</span><br><span class="line"></span><br><span class="line">                if (c == null) &#123;</span><br><span class="line">                    // If still not found, then invoke findClass in order</span><br><span class="line">                    // to find the class.</span><br><span class="line">                    //如果还是找不到，就调用findClass去查找</span><br><span class="line">                    c = findClass(name);</span><br><span class="line">                &#125;</span><br><span class="line">            &#125;</span><br><span class="line">            return c;</span><br><span class="line">    &#125;</span><br><span class="line">    </span><br><span class="line">    private Class&lt;?&gt; findBootstrapClassOrNull(String name)&#123;</span><br><span class="line">        return null;</span><br><span class="line">    &#125;</span><br><span class="line">    </span><br><span class="line">     protected final Class&lt;?&gt; findLoadedClass(String name) &#123;</span><br><span class="line">        ClassLoader loader;</span><br><span class="line">        if (this == BootClassLoader.getInstance())</span><br><span class="line">            loader = null;</span><br><span class="line">        else</span><br><span class="line">            loader = this;</span><br><span class="line">            //通过native方法实现查找</span><br><span class="line">        return VMClassLoader.findLoadedClass(loader, name);</span><br><span class="line">    &#125;</span><br></pre></td></tr></table></figure>
<p>上面类加载的过程就是双亲委派机制。先检查类是否已经被加载，如果已经加载，直接获取并返回。如果没有被加载，parent不为空，则调用parent的loadClass进行加载，依次递归，直到找到或者加载了就返回；如果还没有找到也加载不了，则自己去加载。</p>
<p><strong>BootClassLoader是最后一个加载器</strong>，BootClassLoader重写了findClass和loadClass方法，并且在loadClass方法中不再获取parent，从而结束递归。</p>
<p>在所有parent都不能加载的情况下，DexClassLoader加载过程如下，它的父类继承了BaseDexClassLoader，并重写了findClass方法。</p>
<h5 id="1-3-2-BaseDexClassLoader-findClass"><a href="#1-3-2-BaseDexClassLoader-findClass" class="headerlink" title="1.3.2 BaseDexClassLoader.findClass"></a>1.3.2 BaseDexClassLoader.findClass</h5><p>[-&gt;libcore\dalvik\src\main\java\dalvik\system\BaseDexClassLoader.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br></pre></td><td class="code"><pre><span class="line">@Override</span><br><span class="line">protected Class&lt;?&gt; findClass(String name) throws ClassNotFoundException &#123;</span><br><span class="line">       List&lt;Throwable&gt; suppressedExceptions = new ArrayList&lt;Throwable&gt;();</span><br><span class="line">        //在pathlist中查找指定的class</span><br><span class="line">       Class c = pathList.findClass(name, suppressedExceptions);</span><br><span class="line">       if (c == null) &#123;</span><br><span class="line">           ClassNotFoundException cnfe = new ClassNotFoundException(</span><br><span class="line">                   &quot;Didn&apos;t find class \&quot;&quot; + name + &quot;\&quot; on path: &quot; + pathList);</span><br><span class="line">           for (Throwable t : suppressedExceptions) &#123;</span><br><span class="line">               cnfe.addSuppressed(t);</span><br><span class="line">           &#125;</span><br><span class="line">           throw cnfe;</span><br><span class="line">       &#125;</span><br><span class="line">       return c;</span><br><span class="line">   &#125;</span><br><span class="line">   </span><br><span class="line">  private final DexPathList pathList;</span><br><span class="line">     /**</span><br><span class="line">    * @hide</span><br><span class="line">    */</span><br><span class="line">   public BaseDexClassLoader(String dexPath, File optimizedDirectory,</span><br><span class="line">           String librarySearchPath, ClassLoader parent, boolean isTrusted) &#123;</span><br><span class="line">       super(parent);</span><br><span class="line">       this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);</span><br><span class="line"></span><br><span class="line">       if (reporter != null) &#123;</span><br><span class="line">           reportClassLoaderChain();</span><br><span class="line">       &#125;</span><br><span class="line">   &#125;</span><br></pre></td></tr></table></figure>
<h5 id="1-3-3-DexPathList-findClass"><a href="#1-3-3-DexPathList-findClass" class="headerlink" title="1.3.3 DexPathList.findClass"></a>1.3.3 DexPathList.findClass</h5><p>[-&gt;\libcore\dalvik\src\main\java\dalvik\system\DexPathList.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br></pre></td><td class="code"><pre><span class="line">/**</span><br><span class="line">   * List of dex/resource (class path) elements.</span><br><span class="line">   * Should be called pathElements, but the Facebook app uses reflection</span><br><span class="line">   * to modify &apos;dexElements&apos; (http://b/7726934).</span><br><span class="line">   */</span><br><span class="line">private Element[] dexElements;</span><br><span class="line"></span><br><span class="line">public Class&lt;?&gt; findClass(String name, List&lt;Throwable&gt; suppressed) &#123;</span><br><span class="line">      for (Element element : dexElements) &#123;</span><br><span class="line">         //通过element获取class对象</span><br><span class="line">          Class&lt;?&gt; clazz = element.findClass(name, definingContext, suppressed);</span><br><span class="line">          if (clazz != null) &#123;</span><br><span class="line">              return clazz;</span><br><span class="line">          &#125;</span><br><span class="line">      &#125;</span><br><span class="line"></span><br><span class="line">      if (dexElementsSuppressedExceptions != null) &#123;</span><br><span class="line">          suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));</span><br><span class="line">      &#125;</span><br><span class="line">      return null;</span><br><span class="line">  &#125;</span><br></pre></td></tr></table></figure>
<p>dexElements初始化在构造函数中完成</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br></pre></td><td class="code"><pre><span class="line">DexPathList(ClassLoader definingContext, String dexPath,</span><br><span class="line">           String librarySearchPath, File optimizedDirectory, boolean isTrusted) &#123;</span><br><span class="line">       ...</span><br><span class="line">       this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,</span><br><span class="line">                                          suppressedExceptions, definingContext, isTrusted);</span><br><span class="line"></span><br><span class="line">        ...</span><br><span class="line">   &#125;</span><br></pre></td></tr></table></figure>
<p>生成Element数组，每一个dex对应一个Element</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br><span class="line">43</span><br><span class="line">44</span><br><span class="line">45</span><br><span class="line">46</span><br><span class="line">47</span><br><span class="line">48</span><br><span class="line">49</span><br><span class="line">50</span><br><span class="line">51</span><br><span class="line">52</span><br><span class="line">53</span><br><span class="line">54</span><br><span class="line">55</span><br><span class="line">56</span><br><span class="line">57</span><br><span class="line">58</span><br><span class="line">59</span><br></pre></td><td class="code"><pre><span class="line">private static Element[] makeDexElements(List&lt;File&gt; files, File optimizedDirectory,</span><br><span class="line">           List&lt;IOException&gt; suppressedExceptions, ClassLoader loader, boolean isTrusted) &#123;</span><br><span class="line">     Element[] elements = new Element[files.size()];</span><br><span class="line">     int elementsPos = 0;</span><br><span class="line">     /*</span><br><span class="line">      * Open all files and load the (direct or contained) dex files up front.</span><br><span class="line">      */</span><br><span class="line">     for (File file : files) &#123;</span><br><span class="line">         if (file.isDirectory()) &#123;</span><br><span class="line">             // We support directories for looking up resources. Looking up resources in</span><br><span class="line">             // directories is useful for running libcore tests.</span><br><span class="line">             elements[elementsPos++] = new Element(file);</span><br><span class="line">         &#125; else if (file.isFile()) &#123;</span><br><span class="line">             String name = file.getName();</span><br><span class="line"></span><br><span class="line">             DexFile dex = null;</span><br><span class="line">             if (name.endsWith(DEX_SUFFIX)) &#123;</span><br><span class="line">                 // Raw dex file (not inside a zip/jar).</span><br><span class="line">                 try &#123;</span><br><span class="line">                     dex = loadDexFile(file, optimizedDirectory, loader, elements);</span><br><span class="line">                     if (dex != null) &#123;</span><br><span class="line">                         elements[elementsPos++] = new Element(dex, null);</span><br><span class="line">                     &#125;</span><br><span class="line">                 &#125; catch (IOException suppressed) &#123;</span><br><span class="line">                     System.logE(&quot;Unable to load dex file: &quot; + file, suppressed);</span><br><span class="line">                     suppressedExceptions.add(suppressed);</span><br><span class="line">                 &#125;</span><br><span class="line">             &#125; else &#123;</span><br><span class="line">                 try &#123;</span><br><span class="line">                     dex = loadDexFile(file, optimizedDirectory, loader, elements);</span><br><span class="line">                 &#125; catch (IOException suppressed) &#123;</span><br><span class="line">                     /*</span><br><span class="line">                      * IOException might get thrown &quot;legitimately&quot; by the DexFile constructor if</span><br><span class="line">                      * the zip file turns out to be resource-only (that is, no classes.dex file</span><br><span class="line">                      * in it).</span><br><span class="line">                      * Let dex == null and hang on to the exception to add to the tea-leaves for</span><br><span class="line">                      * when findClass returns null.</span><br><span class="line">                      */</span><br><span class="line">                     suppressedExceptions.add(suppressed);</span><br><span class="line">                 &#125;</span><br><span class="line"></span><br><span class="line">                 if (dex == null) &#123;</span><br><span class="line">                     elements[elementsPos++] = new Element(file);</span><br><span class="line">                 &#125; else &#123;</span><br><span class="line">                     elements[elementsPos++] = new Element(dex, file);</span><br><span class="line">                 &#125;</span><br><span class="line">             &#125;</span><br><span class="line">             if (dex != null &amp;&amp; isTrusted) &#123;</span><br><span class="line">               dex.setTrusted();</span><br><span class="line">             &#125;</span><br><span class="line">         &#125; else &#123;</span><br><span class="line">             System.logW(&quot;ClassLoader referenced unknown path: &quot; + file);</span><br><span class="line">         &#125;</span><br><span class="line">     &#125;</span><br><span class="line">     if (elementsPos != elements.length) &#123;</span><br><span class="line">         elements = Arrays.copyOf(elements, elementsPos);</span><br><span class="line">     &#125;</span><br><span class="line">     return elements;</span><br><span class="line">   &#125;</span><br></pre></td></tr></table></figure>
<p>Class对象从Element中获取，每一个Element对应一个dex文件。通过上面的分析，可以想到插件apk的加载方法，通过DexClassLoader加载器加载插件apk，再通过反射的方法获取到宿主的dexElements，最后将插件dexElements和宿主的dexElements合并并赋值给dexElements，其详细流程如下：</p>
<p>1.创建插件的DexClassLoader类加载器，通过反射获取插件中的dexElements</p>
<p>2.获取宿主的PathClassLoader类加载器，通过反射获取宿主的dexElements</p>
<p>3.合并插件和宿主的dexElements，生成新的Element[]</p>
<p>4.最后通过反射将新的Element[]赋值给宿主的dexElements</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br></pre></td><td class="code"><pre><span class="line">//获取BaseDexClassLoader类的class</span><br><span class="line"> Class&lt;?&gt; clazz = Class.forName(&quot;dalvik.system.BaseDexClassLoader&quot;);</span><br><span class="line"> //获取BaseDexClassLoader类成员变量pathList</span><br><span class="line"> Field pathListField = clazz.getDeclaredField(&quot;pathList&quot;);</span><br><span class="line"> pathListField.setAccessible(true);</span><br><span class="line"></span><br><span class="line"> //获取宿主PathClassLoader</span><br><span class="line"> PathClassLoader pathClassLoader = (PathClassLoader) context.getClassLoader();</span><br><span class="line"> //获取宿主pathlist值</span><br><span class="line"> Object hostPathList = pathListField.get(pathClassLoader);</span><br><span class="line"></span><br><span class="line"> //获取宿主的dexElements</span><br><span class="line"> Class&lt;?&gt; hostPathListClass = hostPathList.getClass();</span><br><span class="line"> Field hostDexElementsField = hostPathListClass.getDeclaredField(&quot;dexElements&quot;);</span><br><span class="line"> hostDexElementsField.setAccessible(true);</span><br><span class="line"> Object[] hostDexElements = (Object[]) hostDexElementsField.get(hostPathList);</span><br><span class="line"></span><br><span class="line"> //获取插件pathlist值</span><br><span class="line"> DexClassLoader dexClassLoader = new DexClassLoader(apkPath,context.getCacheDir().getAbsolutePath(),null,pathClassLoader);</span><br><span class="line"> Object pluginPathList = pathListField.get(dexClassLoader);</span><br><span class="line"></span><br><span class="line"> //获取插件的dexElements</span><br><span class="line"> Class&lt;?&gt; pluginPathListClass = pluginPathList.getClass();</span><br><span class="line"> Field pluginDexElementsField = pluginPathListClass.getDeclaredField(&quot;dexElements&quot;);</span><br><span class="line"> pluginDexElementsField.setAccessible(true);</span><br><span class="line"> Object[] pluginDexElements = (Object[]) pluginDexElementsField.get(pluginPathList);</span><br><span class="line"></span><br><span class="line"> //创建新的Element数组</span><br><span class="line"> Object[] newDexElements = (Object[]) Array.newInstance(hostDexElements.getClass().getComponentType(),hostDexElements.length+pluginDexElements.length);</span><br><span class="line"> System.arraycopy(hostDexElements,0,newDexElements,0,hostDexElements.length);</span><br><span class="line"> System.arraycopy(pluginDexElements,0,newDexElements,hostDexElements.length,pluginDexElements.length);</span><br><span class="line"></span><br><span class="line"> //将新的Element[]赋值给dexElements</span><br><span class="line"> hostDexElementsField.set(hostPathList,newDexElements);</span><br></pre></td></tr></table></figure>
<p>这样就完成了插件apk中类的加载，热修复也用到了这样的类加载的原理，不同的是将插件的Elements放在了最前面。</p>
<h3 id="二、资源加载"><a href="#二、资源加载" class="headerlink" title="二、资源加载"></a>二、资源加载</h3><p>要加载插件中的资源，可以通过AssetManager来实现，因为资源的加载实际上是通过AssetManager来加载的。AssetManager可以通过文件名访问那些被编译过的应用程序的资源文件也可以访问没有被编译过的应用程序资源文件。</p>
<p>下面分析下AssetManager如何加载资源的，获取资源是通过getResources方法：</p>
<p>在ActivityThread中attach 方法里面的createAppContext创建Context时会设置Resources。</p>
<h4 id="2-1-CL-createAppContext"><a href="#2-1-CL-createAppContext" class="headerlink" title="2.1 CL.createAppContext"></a>2.1 CL.createAppContext</h4><p>  [-&gt;base\core\java\android\app\ContextImpl.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><span class="line">static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) &#123;</span><br><span class="line">      if (packageInfo == null) throw new IllegalArgumentException(&quot;packageInfo&quot;);</span><br><span class="line">      ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,</span><br><span class="line">              null);</span><br><span class="line">      context.setResources(packageInfo.getResources());</span><br><span class="line">      return context;</span><br><span class="line">  &#125;</span><br></pre></td></tr></table></figure>
<h4 id="2-2-LoadedApk-getResources"><a href="#2-2-LoadedApk-getResources" class="headerlink" title="2.2 LoadedApk.getResources"></a>2.2 LoadedApk.getResources</h4><p>[-&gt;base\core\java\android\app\LoadedApk.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br></pre></td><td class="code"><pre><span class="line">public Resources getResources() &#123;</span><br><span class="line">     if (mResources == null) &#123;</span><br><span class="line">         final String[] splitPaths;</span><br><span class="line">         try &#123;</span><br><span class="line">             splitPaths = getSplitPaths(null);</span><br><span class="line">         &#125; catch (NameNotFoundException e) &#123;</span><br><span class="line">             // This should never fail.</span><br><span class="line">             throw new AssertionError(&quot;null split not found&quot;);</span><br><span class="line">         &#125;</span><br><span class="line">        //获取ResourceManager对象，调用getResources获取mResources</span><br><span class="line">         mResources = ResourcesManager.getInstance().getResources(null, mResDir,</span><br><span class="line">                 splitPaths, mOverlayDirs, mApplicationInfo.sharedLibraryFiles,</span><br><span class="line">                 Display.DEFAULT_DISPLAY, null, getCompatibilityInfo(),</span><br><span class="line">                 getClassLoader());</span><br><span class="line">     &#125;</span><br><span class="line">     return mResources;</span><br><span class="line"> &#125;</span><br></pre></td></tr></table></figure>
<h4 id="2-3-RM-getResources"><a href="#2-3-RM-getResources" class="headerlink" title="2.3 RM.getResources"></a>2.3 RM.getResources</h4><p>[-&gt;base\core\java\android\app\ResourcesManager.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br></pre></td><td class="code"><pre><span class="line">public @Nullable Resources getResources(@Nullable IBinder activityToken,</span><br><span class="line">          @Nullable String resDir,</span><br><span class="line">          @Nullable String[] splitResDirs,</span><br><span class="line">          @Nullable String[] overlayDirs,</span><br><span class="line">          @Nullable String[] libDirs,</span><br><span class="line">          int displayId,</span><br><span class="line">          @Nullable Configuration overrideConfig,</span><br><span class="line">          @NonNull CompatibilityInfo compatInfo,</span><br><span class="line">          @Nullable ClassLoader classLoader) &#123;</span><br><span class="line">      try &#123;</span><br><span class="line">          Trace.traceBegin(Trace.TRACE_TAG_RESOURCES, &quot;ResourcesManager#getResources&quot;);</span><br><span class="line">          final ResourcesKey key = new ResourcesKey(</span><br><span class="line">                  resDir,</span><br><span class="line">                  splitResDirs,</span><br><span class="line">                  overlayDirs,</span><br><span class="line">                  libDirs,</span><br><span class="line">                  displayId,</span><br><span class="line">                  overrideConfig != null ? new Configuration(overrideConfig) : null, // Copy</span><br><span class="line">                  compatInfo);</span><br><span class="line">          classLoader = classLoader != null ? classLoader : ClassLoader.getSystemClassLoader();</span><br><span class="line">          return getOrCreateResources(activityToken, key, classLoader);</span><br><span class="line">      &#125; finally &#123;</span><br><span class="line">          Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);</span><br><span class="line">      &#125;</span><br><span class="line">  &#125;</span><br></pre></td></tr></table></figure>
<h4 id="2-4-RM-getOrCreateResources"><a href="#2-4-RM-getOrCreateResources" class="headerlink" title="2.4 RM.getOrCreateResources"></a>2.4 RM.getOrCreateResources</h4><p>[-&gt;base\core\java\android\app\ResourcesManager.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br><span class="line">43</span><br><span class="line">44</span><br><span class="line">45</span><br><span class="line">46</span><br><span class="line">47</span><br><span class="line">48</span><br><span class="line">49</span><br><span class="line">50</span><br><span class="line">51</span><br><span class="line">52</span><br><span class="line">53</span><br><span class="line">54</span><br><span class="line">55</span><br><span class="line">56</span><br><span class="line">57</span><br><span class="line">58</span><br><span class="line">59</span><br><span class="line">60</span><br><span class="line">61</span><br><span class="line">62</span><br><span class="line">63</span><br><span class="line">64</span><br><span class="line">65</span><br><span class="line">66</span><br><span class="line">67</span><br><span class="line">68</span><br><span class="line">69</span><br><span class="line">70</span><br><span class="line">71</span><br><span class="line">72</span><br></pre></td><td class="code"><pre><span class="line">private @Nullable Resources getOrCreateResources(@Nullable IBinder activityToken,</span><br><span class="line">           @NonNull ResourcesKey key, @NonNull ClassLoader classLoader) &#123;</span><br><span class="line">       synchronized (this) &#123;</span><br><span class="line">           if (DEBUG) &#123;</span><br><span class="line">               Throwable here = new Throwable();</span><br><span class="line">               here.fillInStackTrace();</span><br><span class="line">               Slog.w(TAG, &quot;!! Get resources for activity=&quot; + activityToken + &quot; key=&quot; + key, here);</span><br><span class="line">           &#125;</span><br><span class="line"></span><br><span class="line">           if (activityToken != null) &#123;</span><br><span class="line">               final ActivityResources activityResources =</span><br><span class="line">                       getOrCreateActivityResourcesStructLocked(activityToken);</span><br><span class="line"></span><br><span class="line">               // Clean up any dead references so they don&apos;t pile up.</span><br><span class="line">               ArrayUtils.unstableRemoveIf(activityResources.activityResources,</span><br><span class="line">                       sEmptyReferencePredicate);</span><br><span class="line"></span><br><span class="line">               // Rebase the key&apos;s override config on top of the Activity&apos;s base override.</span><br><span class="line">               if (key.hasOverrideConfiguration()</span><br><span class="line">                       &amp;&amp; !activityResources.overrideConfig.equals(Configuration.EMPTY)) &#123;</span><br><span class="line">                   final Configuration temp = new Configuration(activityResources.overrideConfig);</span><br><span class="line">                   temp.updateFrom(key.mOverrideConfiguration);</span><br><span class="line">                   key.mOverrideConfiguration.setTo(temp);</span><br><span class="line">               &#125;</span><br><span class="line"></span><br><span class="line">               ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);</span><br><span class="line">               if (resourcesImpl != null) &#123;</span><br><span class="line">                   if (DEBUG) &#123;</span><br><span class="line">                       Slog.d(TAG, &quot;- using existing impl=&quot; + resourcesImpl);</span><br><span class="line">                   &#125;</span><br><span class="line">                   return getOrCreateResourcesForActivityLocked(activityToken, classLoader,</span><br><span class="line">                           resourcesImpl, key.mCompatInfo);</span><br><span class="line">               &#125;</span><br><span class="line"></span><br><span class="line">               // We will create the ResourcesImpl object outside of holding this lock.</span><br><span class="line"></span><br><span class="line">           &#125; else &#123;</span><br><span class="line">               // Clean up any dead references so they don&apos;t pile up.</span><br><span class="line">               ArrayUtils.unstableRemoveIf(mResourceReferences, sEmptyReferencePredicate);</span><br><span class="line"></span><br><span class="line">               // Not tied to an Activity, find a shared Resources that has the right ResourcesImpl</span><br><span class="line">               ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);</span><br><span class="line">               if (resourcesImpl != null) &#123;</span><br><span class="line">                   if (DEBUG) &#123;</span><br><span class="line">                       Slog.d(TAG, &quot;- using existing impl=&quot; + resourcesImpl);</span><br><span class="line">                   &#125;</span><br><span class="line">                   return getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);</span><br><span class="line">               &#125;</span><br><span class="line"></span><br><span class="line">               // We will create the ResourcesImpl object outside of holding this lock.</span><br><span class="line">           &#125;</span><br><span class="line"></span><br><span class="line">           // If we&apos;re here, we didn&apos;t find a suitable ResourcesImpl to use, so create one now.</span><br><span class="line">           //创建ResourcesImpl对象</span><br><span class="line">           ResourcesImpl resourcesImpl = createResourcesImpl(key);</span><br><span class="line">           if (resourcesImpl == null) &#123;</span><br><span class="line">               return null;</span><br><span class="line">           &#125;</span><br><span class="line"></span><br><span class="line">           // Add this ResourcesImpl to the cache.</span><br><span class="line">           mResourceImpls.put(key, new WeakReference&lt;&gt;(resourcesImpl));</span><br><span class="line"></span><br><span class="line">           final Resources resources;</span><br><span class="line">           if (activityToken != null) &#123;</span><br><span class="line">               resources = getOrCreateResourcesForActivityLocked(activityToken, classLoader,</span><br><span class="line">                       resourcesImpl, key.mCompatInfo);</span><br><span class="line">           &#125; else &#123;</span><br><span class="line">               resources = getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);</span><br><span class="line">           &#125;</span><br><span class="line">           return resources;</span><br><span class="line">       &#125;</span><br><span class="line">   &#125;</span><br></pre></td></tr></table></figure>
<h4 id="2-5-RM-createResourcesImpl"><a href="#2-5-RM-createResourcesImpl" class="headerlink" title="2.5 RM.createResourcesImpl"></a>2.5 RM.createResourcesImpl</h4><p>[-&gt;base\core\java\android\app\ResourcesManager.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br></pre></td><td class="code"><pre><span class="line">private @Nullable ResourcesImpl createResourcesImpl(@NonNull ResourcesKey key) &#123;</span><br><span class="line">       final DisplayAdjustments daj = new DisplayAdjustments(key.mOverrideConfiguration);</span><br><span class="line">       daj.setCompatibilityInfo(key.mCompatInfo);</span><br><span class="line">       //创建AssetManager</span><br><span class="line">       final AssetManager assets = createAssetManager(key);</span><br><span class="line">       if (assets == null) &#123;</span><br><span class="line">           return null;</span><br><span class="line">       &#125;</span><br><span class="line"></span><br><span class="line">       final DisplayMetrics dm = getDisplayMetrics(key.mDisplayId, daj);</span><br><span class="line">       final Configuration config = generateConfig(key, dm);</span><br><span class="line">       //将assets对象传入到ResourcesImpl中</span><br><span class="line">       final ResourcesImpl impl = new ResourcesImpl(assets, dm, config, daj);</span><br><span class="line"></span><br><span class="line">       if (DEBUG) &#123;</span><br><span class="line">           Slog.d(TAG, &quot;- creating impl=&quot; + impl + &quot; with key: &quot; + key);</span><br><span class="line">       &#125;</span><br><span class="line">       return impl;</span><br><span class="line">   &#125;</span><br></pre></td></tr></table></figure>
<h4 id="2-6-RM-createAssetManager"><a href="#2-6-RM-createAssetManager" class="headerlink" title="2.6 RM.createAssetManager"></a>2.6 RM.createAssetManager</h4><p>[-&gt;base\core\java\android\app\ResourcesManager.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br><span class="line">43</span><br><span class="line">44</span><br><span class="line">45</span><br><span class="line">46</span><br><span class="line">47</span><br><span class="line">48</span><br><span class="line">49</span><br><span class="line">50</span><br><span class="line">51</span><br><span class="line">52</span><br><span class="line">53</span><br><span class="line">54</span><br><span class="line">55</span><br><span class="line">56</span><br><span class="line">57</span><br><span class="line">58</span><br><span class="line">59</span><br><span class="line">60</span><br><span class="line">61</span><br></pre></td><td class="code"><pre><span class="line">protected @Nullable AssetManager createAssetManager(@NonNull final ResourcesKey key) &#123;</span><br><span class="line">       final AssetManager.Builder builder = new AssetManager.Builder();</span><br><span class="line"></span><br><span class="line">       // resDir can be null if the &apos;android&apos; package is creating a new Resources object.</span><br><span class="line">       // This is fine, since each AssetManager automatically loads the &apos;android&apos; package</span><br><span class="line">       // already.</span><br><span class="line">       if (key.mResDir != null) &#123;</span><br><span class="line">           try &#123;</span><br><span class="line">               builder.addApkAssets(loadApkAssets(key.mResDir, false /*sharedLib*/,</span><br><span class="line">                       false /*overlay*/));</span><br><span class="line">           &#125; catch (IOException e) &#123;</span><br><span class="line">               Log.e(TAG, &quot;failed to add asset path &quot; + key.mResDir);</span><br><span class="line">               return null;</span><br><span class="line">           &#125;</span><br><span class="line">       &#125;</span><br><span class="line"></span><br><span class="line">       if (key.mSplitResDirs != null) &#123;</span><br><span class="line">           for (final String splitResDir : key.mSplitResDirs) &#123;</span><br><span class="line">               try &#123;</span><br><span class="line">                   builder.addApkAssets(loadApkAssets(splitResDir, false /*sharedLib*/,</span><br><span class="line">                           false /*overlay*/));</span><br><span class="line">               &#125; catch (IOException e) &#123;</span><br><span class="line">                   Log.e(TAG, &quot;failed to add split asset path &quot; + splitResDir);</span><br><span class="line">                   return null;</span><br><span class="line">               &#125;</span><br><span class="line">           &#125;</span><br><span class="line">       &#125;</span><br><span class="line"></span><br><span class="line">       if (key.mOverlayDirs != null) &#123;</span><br><span class="line">           for (final String idmapPath : key.mOverlayDirs) &#123;</span><br><span class="line">               try &#123;</span><br><span class="line">                   builder.addApkAssets(loadApkAssets(idmapPath, false /*sharedLib*/,</span><br><span class="line">                           true /*overlay*/));</span><br><span class="line">               &#125; catch (IOException e) &#123;</span><br><span class="line">                   Log.w(TAG, &quot;failed to add overlay path &quot; + idmapPath);</span><br><span class="line"></span><br><span class="line">                   // continue.</span><br><span class="line">               &#125;</span><br><span class="line">           &#125;</span><br><span class="line">       &#125;</span><br><span class="line"></span><br><span class="line">       if (key.mLibDirs != null) &#123;</span><br><span class="line">           for (final String libDir : key.mLibDirs) &#123;</span><br><span class="line">               if (libDir.endsWith(&quot;.apk&quot;)) &#123;</span><br><span class="line">                   // Avoid opening files we know do not have resources,</span><br><span class="line">                   // like code-only .jar files.</span><br><span class="line">                   try &#123;</span><br><span class="line">                       builder.addApkAssets(loadApkAssets(libDir, true /*sharedLib*/,</span><br><span class="line">                               false /*overlay*/));</span><br><span class="line">                   &#125; catch (IOException e) &#123;</span><br><span class="line">                       Log.w(TAG, &quot;Asset path &apos;&quot; + libDir +</span><br><span class="line">                               &quot;&apos; does not exist or contains no resources.&quot;);</span><br><span class="line"></span><br><span class="line">                       // continue.</span><br><span class="line">                   &#125;</span><br><span class="line">               &#125;</span><br><span class="line">           &#125;</span><br><span class="line">       &#125;</span><br><span class="line">       //最后通过build完成创建</span><br><span class="line">       return builder.build();</span><br><span class="line">   &#125;</span><br></pre></td></tr></table></figure>
<h4 id="2-7-AssetManager-addApkAssets"><a href="#2-7-AssetManager-addApkAssets" class="headerlink" title="2.7 AssetManager.addApkAssets"></a>2.7 AssetManager.addApkAssets</h4><p>[-&gt;base\core\java\android\content\res\AssetManager.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br></pre></td><td class="code"><pre><span class="line">public Builder addApkAssets(ApkAssets apkAssets) &#123;</span><br><span class="line">          mUserApkAssets.add(apkAssets);</span><br><span class="line">          return this;</span><br><span class="line">&#125;</span><br><span class="line"> public AssetManager build() &#123;</span><br><span class="line">          // Retrieving the system ApkAssets forces their creation as well.</span><br><span class="line">          final ApkAssets[] systemApkAssets = getSystem().getApkAssets();</span><br><span class="line"></span><br><span class="line">          final int totalApkAssetCount = systemApkAssets.length + mUserApkAssets.size();</span><br><span class="line">          final ApkAssets[] apkAssets = new ApkAssets[totalApkAssetCount];</span><br><span class="line"></span><br><span class="line">          System.arraycopy(systemApkAssets, 0, apkAssets, 0, systemApkAssets.length);</span><br><span class="line"></span><br><span class="line">          final int userApkAssetCount = mUserApkAssets.size();</span><br><span class="line">          for (int i = 0; i &lt; userApkAssetCount; i++) &#123;</span><br><span class="line">              apkAssets[i + systemApkAssets.length] = mUserApkAssets.get(i);</span><br><span class="line">          &#125;</span><br><span class="line"></span><br><span class="line">          // Calling this constructor prevents creation of system ApkAssets, which we took care</span><br><span class="line">          // of in this Builder.</span><br><span class="line">          final AssetManager assetManager = new AssetManager(false /*sentinel*/);</span><br><span class="line">          assetManager.mApkAssets = apkAssets;</span><br><span class="line">          AssetManager.nativeSetApkAssets(assetManager.mObject, apkAssets,</span><br><span class="line">                  false /*invalidateCaches*/);</span><br><span class="line">          return assetManager;</span><br><span class="line">      &#125;</span><br></pre></td></tr></table></figure>
<h4 id="2-8-RM-loadApkAssets"><a href="#2-8-RM-loadApkAssets" class="headerlink" title="2.8 RM.loadApkAssets"></a>2.8 RM.loadApkAssets</h4><p>[-&gt;base\core\java\android\app\ResourcesManager.java]</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br></pre></td><td class="code"><pre><span class="line">private @NonNull ApkAssets loadApkAssets(String path, boolean sharedLib, boolean overlay)</span><br><span class="line">           throws IOException &#123;</span><br><span class="line">       final ApkKey newKey = new ApkKey(path, sharedLib, overlay);</span><br><span class="line">       ApkAssets apkAssets = null;</span><br><span class="line">       if (mLoadedApkAssets != null) &#123;</span><br><span class="line">           apkAssets = mLoadedApkAssets.get(newKey);</span><br><span class="line">           if (apkAssets != null) &#123;</span><br><span class="line">               return apkAssets;</span><br><span class="line">           &#125;</span><br><span class="line">       &#125;</span><br><span class="line"></span><br><span class="line">       // Optimistically check if this ApkAssets exists somewhere else.</span><br><span class="line">       final WeakReference&lt;ApkAssets&gt; apkAssetsRef = mCachedApkAssets.get(newKey);</span><br><span class="line">       if (apkAssetsRef != null) &#123;</span><br><span class="line">           apkAssets = apkAssetsRef.get();</span><br><span class="line">           if (apkAssets != null) &#123;</span><br><span class="line">               if (mLoadedApkAssets != null) &#123;</span><br><span class="line">                   mLoadedApkAssets.put(newKey, apkAssets);</span><br><span class="line">               &#125;</span><br><span class="line"></span><br><span class="line">               return apkAssets;</span><br><span class="line">           &#125; else &#123;</span><br><span class="line">               // Clean up the reference.</span><br><span class="line">               mCachedApkAssets.remove(newKey);</span><br><span class="line">           &#125;</span><br><span class="line">       &#125;</span><br><span class="line"></span><br><span class="line">       // We must load this from disk.</span><br><span class="line">       if (overlay) &#123;</span><br><span class="line">           apkAssets = ApkAssets.loadOverlayFromPath(overlayPathToIdmapPath(path),</span><br><span class="line">                   false /*system*/);</span><br><span class="line">       &#125; else &#123;</span><br><span class="line">           apkAssets = ApkAssets.loadFromPath(path, false /*system*/, sharedLib);</span><br><span class="line">       &#125;</span><br><span class="line"></span><br><span class="line">       if (mLoadedApkAssets != null) &#123;</span><br><span class="line">           mLoadedApkAssets.put(newKey, apkAssets);</span><br><span class="line">       &#125;</span><br><span class="line"></span><br><span class="line">       mCachedApkAssets.put(newKey, new WeakReference&lt;&gt;(apkAssets));</span><br><span class="line">       return apkAssets;</span><br><span class="line">   &#125;</span><br></pre></td></tr></table></figure>
<p>按照这样的流程比较难hook，看下之前添加的方法addAssetPath，这个方法已经被废弃，但还可以用，hook这个方法比较简单。</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br></pre></td><td class="code"><pre><span class="line">/**</span><br><span class="line">    * @deprecated Use &#123;@link #setApkAssets(ApkAssets[], boolean)&#125;</span><br><span class="line">    * @hide</span><br><span class="line">    */</span><br><span class="line">   @Deprecated</span><br><span class="line">   @UnsupportedAppUsage</span><br><span class="line">   public int addAssetPath(String path) &#123;</span><br><span class="line">       return addAssetPathInternal(path, false /*overlay*/, false /*appAsLib*/);</span><br><span class="line">   &#125;</span><br></pre></td></tr></table></figure>
<p>1.创建一个AssetManager对象，并调用addAssetPath方法，将插件apk路径作为参数传入</p>
<p>2.将创建的AssetManager对象作为参数，创建一个新的Resourced对象，并返回给插件使用。</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br></pre></td><td class="code"><pre><span class="line">public static Resources loadResource(Context context)&#123;</span><br><span class="line">       try &#123;</span><br><span class="line"></span><br><span class="line">           AssetManager assetManager = AssetManager.class.newInstance();</span><br><span class="line">           Method addAssetPathMethod = AssetManager.class.getDeclaredMethod(&quot;addAssetPath&quot;,String.class);</span><br><span class="line">           //加载插件资源</span><br><span class="line">           addAssetPathMethod.invoke(assetManager,apkPath);</span><br><span class="line">           Resources resources = context.getResources();</span><br><span class="line">           return new Resources(assetManager,resources.getDisplayMetrics(),resources.getConfiguration());</span><br><span class="line">       &#125;catch (Exception e)&#123;</span><br><span class="line">           e.printStackTrace();</span><br><span class="line">       &#125;</span><br><span class="line">       return null;</span><br><span class="line">   &#125;</span><br></pre></td></tr></table></figure>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br></pre></td><td class="code"><pre><span class="line">private Resources resources;</span><br><span class="line">@Override</span><br><span class="line">public void onCreate() &#123;</span><br><span class="line">    super.onCreate();</span><br><span class="line">    //加载插件类</span><br><span class="line">    LoadClassUtil.loadClass(this);</span><br><span class="line">    //创建新的resources</span><br><span class="line">    resources = LoadClassUtil.loadResource(this);</span><br><span class="line">    HookUtil.hookAMS();</span><br><span class="line">    HookUtil.hookHandler();</span><br><span class="line">&#125;</span><br><span class="line"></span><br><span class="line">@Override</span><br><span class="line">public Resources getResources() &#123;</span><br><span class="line">   //resources为空，相当于没有重写，不为空是返回新建的resources对象</span><br><span class="line">    return resources==null?super.getResources():resources;</span><br><span class="line">&#125;</span><br></pre></td></tr></table></figure>
<p>然后让插件的Activity重写getResources方法，获取的资源就是新创建的resources对象</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><span class="line">@Override</span><br><span class="line">  public Resources getResources() &#123;</span><br><span class="line">      if(getApplication()!=null&amp;&amp;getApplication()!=null)&#123;</span><br><span class="line">          return getApplication().getResources();</span><br><span class="line">      &#125;</span><br><span class="line">      return super.getResources();</span><br><span class="line">  &#125;</span><br></pre></td></tr></table></figure>
<h3 id="三、hook-Activity"><a href="#三、hook-Activity" class="headerlink" title="三、hook Activity"></a>三、hook Activity</h3><p>首先在宿主里面创建一个ProxyActivity，并且在Manifest中注册（如果需要对应不同启动模式的Activity，可以全部把每种启动模式下的Activity都注册）。当启动插件Activity时，进入AMS之前，通过Hook将插件Activity替换成ProxyActivity，在进入AMS之后，通过handler发送消息时使用hook将ProxyActivity替换成插件的Activity。</p>
<p>startActivity的流程如下图，可以看到这两个具体的hook点</p>
<p><img src="/2020/Android10.0如何hook Activity/startActivity.jpg" alt="startActivity" style="zoom: 50%;"></p>
<h4 id="3-1-hook-AMS"><a href="#3-1-hook-AMS" class="headerlink" title="3.1 hook AMS"></a>3.1 hook AMS</h4><p>在进入到AMS之前，这个是最后一步，那么如何将intent换成插件的intent的呢？可以通过动态代理实现。</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br></pre></td><td class="code"><pre><span class="line">public ActivityResult execStartActivity(</span><br><span class="line">      Context who, IBinder contextThread, IBinder token, String target,</span><br><span class="line">      Intent intent, int requestCode, Bundle options) &#123;</span><br><span class="line">      ...</span><br><span class="line">      try &#123;</span><br><span class="line">          intent.migrateExtraStreamToClipData();</span><br><span class="line">          intent.prepareToLeaveProcess(who);</span><br><span class="line">          int result = ActivityManager.getService()</span><br><span class="line">              .startActivity(whoThread, who.getBasePackageName(), intent,</span><br><span class="line">                      intent.resolveTypeIfNeeded(who.getContentResolver()),</span><br><span class="line">                      token, target, requestCode, 0, null, options);</span><br><span class="line">          checkStartActivityResult(result, intent);</span><br><span class="line">      &#125; catch (RemoteException e) &#123;</span><br><span class="line">          throw new RuntimeException(&quot;Failure from system&quot;, e);</span><br><span class="line">      &#125;</span><br><span class="line">      return null;</span><br><span class="line">  &#125;</span><br></pre></td></tr></table></figure>
<p>ActivityManager.getService具体实现如下</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br></pre></td><td class="code"><pre><span class="line">/**</span><br><span class="line">    * @hide</span><br><span class="line">    */</span><br><span class="line">   @UnsupportedAppUsage</span><br><span class="line">   public static IActivityManager getService() &#123;</span><br><span class="line">       return IActivityManagerSingleton.get();</span><br><span class="line">   &#125;</span><br><span class="line">   public abstract class Singleton&lt;T&gt; &#123;</span><br><span class="line">   @UnsupportedAppUsage</span><br><span class="line">   private T mInstance;</span><br><span class="line"></span><br><span class="line">   protected abstract T create();</span><br><span class="line"></span><br><span class="line">   @UnsupportedAppUsage</span><br><span class="line">   public final T get() &#123;</span><br><span class="line">       synchronized (this) &#123;</span><br><span class="line">           if (mInstance == null) &#123;</span><br><span class="line">               mInstance = create();</span><br><span class="line">           &#125;</span><br><span class="line">           return mInstance;</span><br><span class="line">       &#125;</span><br><span class="line">   &#125;</span><br><span class="line"> &#125;  </span><br><span class="line">   @UnsupportedAppUsage</span><br><span class="line">   private static final Singleton&lt;IActivityManager&gt; IActivityManagerSingleton =</span><br><span class="line">           new Singleton&lt;IActivityManager&gt;() &#123;</span><br><span class="line">               @Override</span><br><span class="line">               protected IActivityManager create() &#123;</span><br><span class="line">                   final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);</span><br><span class="line">                   //获取AMS的代理</span><br><span class="line">                   final IActivityManager am = IActivityManager.Stub.asInterface(b);</span><br><span class="line">                   return am;</span><br><span class="line">               &#125;</span><br><span class="line">           &#125;;</span><br></pre></td></tr></table></figure>
<p>第一步获取IActivityManager对象</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br></pre></td><td class="code"><pre><span class="line">//获取IActivityManagerSingleton对象，用于获取mIntance</span><br><span class="line"> Class&lt;?&gt; clazz = Class.forName(&quot;android.app.ActivityManager&quot;);</span><br><span class="line"> Field singletonFiled = clazz.getDeclaredField(&quot;IActivityManagerSingleton&quot;);</span><br><span class="line"> singletonFiled.setAccessible(true);</span><br><span class="line"> Object singleton = singletonFiled.get(null);</span><br><span class="line"></span><br><span class="line"> //获取Singleton对象</span><br><span class="line"> Class&lt;?&gt; singletonClass = Class.forName(&quot;android.util.Singleton&quot;);</span><br><span class="line"> Field mInstanceField = singletonClass.getDeclaredField(&quot;mInstance&quot;);</span><br><span class="line"> mInstanceField.setAccessible(true);</span><br><span class="line"> //获取mInstance对象</span><br><span class="line"> final Object mInstance = mInstanceField.get(singleton);</span><br></pre></td></tr></table></figure>
<p>第二步将intent替换成插件的intent</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br></pre></td><td class="code"><pre><span class="line"> Class&lt;?&gt; iActivityManagerClass = Class.forName(&quot;android.app.IActivityManager&quot;);</span><br><span class="line">          Object proxyInstance = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), new Class[]&#123;iActivityManagerClass&#125;,</span><br><span class="line">                  new InvocationHandler() &#123;</span><br><span class="line">                      @Override</span><br><span class="line">                      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable &#123;</span><br><span class="line">                          if(&quot;startActivity&quot;.equals(method.getName()))&#123;</span><br><span class="line">                              int index = 0;</span><br><span class="line">                              for(int i=0;i&lt;args.length;i++)&#123;</span><br><span class="line">                                  if(args[i] instanceof Intent)&#123;</span><br><span class="line">                                      index = i;</span><br><span class="line">                                      break;</span><br><span class="line">                                  &#125;</span><br><span class="line">                              &#125;</span><br><span class="line">                              //启动插件的intent</span><br><span class="line">                              Intent intent = (Intent) args[index];</span><br><span class="line">                              //启动代理的intent</span><br><span class="line">                              Intent proxyIntent = new Intent();</span><br><span class="line">                   proxyIntent.setClassName(&quot;com.android.test&quot;,&quot;com.android.test.ProxyActivity&quot;);</span><br><span class="line">                              //保存插件的intent</span><br><span class="line">                              proxyIntent.putExtra(&quot;target&quot;,intent);</span><br><span class="line">                              args[index] = proxyIntent;</span><br><span class="line">                          &#125;</span><br><span class="line">                          return method.invoke(mInstance,args);</span><br><span class="line">                      &#125;</span><br><span class="line">                  &#125;);</span><br><span class="line">//替换mInstance对象</span><br><span class="line">mInstanceField.set(singleton,proxyInstance);</span><br></pre></td></tr></table></figure>
<p>这样第一个hook点就完成了，下面看下第二步hook点</p>
<h4 id="3-2-hook-Handler"><a href="#3-2-hook-Handler" class="headerlink" title="3.2 hook Handler"></a>3.2 hook Handler</h4><p>在启动Activity之前的操作，Android10.0用状态模式完成Activity的生命周期的启动，那么如何将代理的intent替换成插件的intent的呢？从源码可以看出最后通过scheduleTransaction方法启动Activity，那么是否通过clientTransaction替换intent的呢，通过分析startActivity的启动流程，答案是可以的。</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br></pre></td><td class="code"><pre><span class="line">//创建Activity启动事务</span><br><span class="line">// Create activity launch transaction.</span><br><span class="line">final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,</span><br><span class="line">        r.appToken);</span><br><span class="line">clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),</span><br><span class="line">        System.identityHashCode(r), r.info,</span><br><span class="line">        // TODO: Have this take the merged configuration instead of separate global</span><br><span class="line">        // and override configs.</span><br><span class="line">        mergedConfiguration.getGlobalConfiguration(),</span><br><span class="line">        mergedConfiguration.getOverrideConfiguration(), r.compat,</span><br><span class="line">        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,</span><br><span class="line">        r.persistentState, results, newIntents, mService.isNextTransitionForward(),</span><br><span class="line">        profilerInfo));</span><br><span class="line"></span><br><span class="line">//设置目标事务的状态为onResume</span><br><span class="line">// Set desired final state.</span><br><span class="line">final ActivityLifecycleItem lifecycleItem;</span><br><span class="line">if (andResume) &#123;</span><br><span class="line">    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());</span><br><span class="line">&#125; else &#123;</span><br><span class="line">    lifecycleItem = PauseActivityItem.obtain();</span><br><span class="line">&#125;</span><br><span class="line">clientTransaction.setLifecycleStateRequest(lifecycleItem);</span><br><span class="line"></span><br><span class="line">//通过transaciton方式开始activity生命周期，onCreate,onStart,onResume</span><br><span class="line">// Schedule transaction.</span><br><span class="line">mService.getLifecycleManager().scheduleTransaction(clientTransaction);</span><br></pre></td></tr></table></figure>
<p>scheduleTransaction还是要通过Handler发送消息，进入EXECUTE_TRANSACTION分支</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br></pre></td><td class="code"><pre><span class="line">public void handleMessage(Message msg) &#123;</span><br><span class="line">         ...</span><br><span class="line">         case EXECUTE_TRANSACTION:</span><br><span class="line">                    final ClientTransaction transaction = (ClientTransaction) msg.obj;</span><br><span class="line">                    mTransactionExecutor.execute(transaction);</span><br><span class="line">                    if (isSystem()) &#123;</span><br><span class="line">                        // Client transactions inside system process are recycled on the client side</span><br><span class="line">                        // instead of ClientLifecycleManager to avoid being cleared before this</span><br><span class="line">                        // message is handled.</span><br><span class="line">                        transaction.recycle();</span><br><span class="line">                    &#125;</span><br><span class="line">                    // TODO(lifecycler): Recycle locally scheduled transactions.</span><br><span class="line">                    break;</span><br><span class="line">           ....</span><br><span class="line">     &#125;</span><br></pre></td></tr></table></figure>
<p>那么要替换intent首先需要hook handleMessage</p>
<figure class="highlight plain"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br><span class="line">43</span><br></pre></td><td class="code"><pre><span class="line">//获取ActivityThread对象</span><br><span class="line">Class&lt;?&gt; clazz = Class.forName(&quot;android.app.ActivityThread&quot;);</span><br><span class="line">Field sCurrentActivityThreadField = clazz.getDeclaredField(&quot;sCurrentActivityThread&quot;);</span><br><span class="line">sCurrentActivityThreadField.setAccessible(true);</span><br><span class="line">Object activityThread = sCurrentActivityThreadField.get(null);</span><br><span class="line"></span><br><span class="line">//获取handler</span><br><span class="line">Field mHField = clazz.getDeclaredField(&quot;mH&quot;);</span><br><span class="line">mHField.setAccessible(true);</span><br><span class="line">Object mH = mHField.get(activityThread);</span><br><span class="line"></span><br><span class="line">//获取callback对象</span><br><span class="line">Field mCallbackField = Handler.class.getDeclaredField(&quot;mCallback&quot;);</span><br><span class="line">mCallbackField.setAccessible(true);</span><br><span class="line">mCallbackField.set(mH, new Handler.Callback() &#123;</span><br><span class="line">    @Override</span><br><span class="line">    public boolean handleMessage(Message msg) &#123;</span><br><span class="line">      switch (msg.what)&#123;</span><br><span class="line">          case 159:</span><br><span class="line">              try &#123;</span><br><span class="line">                  //获取List&lt;ClientTransactionItem&gt; mActivityCallbacks对象</span><br><span class="line">                  Class&lt;?&gt; clientTransactionClass = msg.obj.getClass();</span><br><span class="line">                  Field mActivityCallbacksField = clientTransactionClass.getDeclaredField(&quot;mActivityCallbacks&quot;);</span><br><span class="line">                  mActivityCallbacksField.setAccessible(true);</span><br><span class="line">                  List mActivityCallbacks = (List) mActivityCallbacksField.get(msg.obj);</span><br><span class="line">                  for(int i=0;i&lt;mActivityCallbacks.size();i++)&#123;                          if(mActivityCallbacks.get(i).getClass().getName().equals(&quot;android.app.servertransaction.LaunchActivityItem&quot;))&#123;</span><br><span class="line">                          //获取LaunchActivityItem</span><br><span class="line">                          Object launchActivityItem = mActivityCallbacks.get(i);</span><br><span class="line">                          Class&lt;?&gt; launchActivityItemClass = launchActivityItem.getClass();</span><br><span class="line"></span><br><span class="line">                          Field mIntentField = launchActivityItemClass.getDeclaredField(&quot;mIntent&quot;);</span><br><span class="line">                          mIntentField.setAccessible(true);</span><br><span class="line"></span><br><span class="line">                          //代理intent</span><br><span class="line">                          Intent proxyIntent = (Intent) mIntentField.get(launchActivityItem);</span><br><span class="line">                          //插件intent</span><br><span class="line">                          Intent intent = proxyIntent.getParcelableExtra(&quot;target&quot;);</span><br><span class="line">                          //插件intent替换代理intent</span><br><span class="line">                          proxyIntent.setComponent(intent.getComponent());</span><br><span class="line"></span><br><span class="line">                      &#125;</span><br><span class="line"></span><br><span class="line">                  &#125;</span><br></pre></td></tr></table></figure>
<p>hook了handleMessage后，通过LaunchActivityItem中的mIntent完成代理intent的替换。</p>
<h3 id="四、总结"><a href="#四、总结" class="headerlink" title="四、总结"></a>四、总结</h3><p>hook Activity涉及的技术比较多，Activity的启动流程，类加载，动态代理，资源加载，反射，binder机制等，只有在掌握了这些技术的基础上才能够完成插件化技术的开发。</p>
<p>本文介绍了hook Activity的具体实现方法，通过该方法引申出类的加载和资源加载的原理，并分析具体插件的加载实现，后面结合之前文章分析的<a href="https://skytoby.github.io/2019/startActivity%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/" target="_blank" rel="noopener">startActivity的启动流程</a>，引出了两个具体的hook点：</p>
<p>1.在ActivityManager.getService().startActivity时通过反射获取IActivityManager对象，startActivity时将插件的Activity替换成代理的Activity；</p>
<p>2.反射获取ActivityThread对象，通过获取类成员变量mH，重新设置callback，将handleMessage中的EXECUTE_TRANSACTION分支进行hook，在mActivityCallbacks中找到LaunchActivityItem，将其类成员变量mIntent替换成插件的Activity，这样就完成的插件Activity的启动过程。</p>