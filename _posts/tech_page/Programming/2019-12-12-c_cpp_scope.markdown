---
layout: tech_post
title:  " C/C++ Scope, Storage, and Linkage"
date:   2019-12-12 11:51:36 -0300
catalogue: Programming
tags: C++ 
description: Notes for C/C++ identifier scope, storage, and linkage.
chinese_link: /chinese/C_Cpp_scope.html
---
## Table of Content
* TOC
{:toc}

Identifier Scope, Storage Class, and Linkage are three key attributes of variables and functions.
Scope, storage, and linkage are decided by definition place and identifiers.

   | Definition  |  Storage   |  Active regions  |  Linkage   |
   |  ---        | ----       |  ----            | ----       |
   | in function blocks      | auto    |       in blocks  |     NO       | 
   | outside functions       | static  |       in the file |          internal |
   | With  <span style="color:red;">'static'</span> in blocks  | static  |       in blocks |          NO |
   | With  <span style="color:red;">'static'</span> outside function  | static  |       in the file |          internal |
   | With  <span style="color:red;">'extern'</span> in blocks  | static  |       in blocks |          NO |
   | With  <span style="color:red;">'extern'</span> outside function  | static  |       in blocks |          external |

   static, 
   
   external can be accessed in other files, internal can be accessed in current file, NO can be accessed in current block or function.  


## Scope



## storage 

### static 


