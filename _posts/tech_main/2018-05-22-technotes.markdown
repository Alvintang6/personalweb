---
layout: tech_menu
title: "Notes for presonal use"
date:   2018-05-22 17:51:36 -0400
tags: 
topics: Notes
---
<SCRIPT LANGUAGE="JavaScript">
            function password() 
            {
                var testV = 1;
                var pass1 = prompt('press password:','');
                while (testV < 3) 
                {
                    if (!pass1)
                    history.go(-1);
                    if (pass1 == "7654321") 
                    {
                        alert('right!');
                        break;
                    }
                    testV+=1;
                    var pass1 =
                    prompt('wrong:');
                }
                if (pass1!="password" & testV ==3)
                history.go(-1);
                return " ";
            }
            document.write(password());
        </SCRIPT>

个人在线笔记						

