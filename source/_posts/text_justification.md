---
layout: post
title: Text Justification
date: 2016-02-06
tags: leetcode
categories: Exercise
---


Given an array of words and a length L, format the text such that each line has exactly L characters and is fully (left and right) justified.

You should pack your words in a greedy approach; that is, pack as many words as you can in each line. Pad extra spaces ' ' \\when necessary so that each line has exactly L characters.

Extra spaces between words should be distributed as evenly as possible. If the number of spaces on a line do not divide \\evenly between words, the empty slots on the left will be assigned more spaces than the slots on the right.

For the last line of text, it should be left justified and no extra space is inserted between words.

For example,
words: ["This", "is", "an", "example", "of", "text", "justification."]
L: 16.

```
public class Solution {
    public static List<String> fullJustify(String[] words, int maxWidth) {
        List<String> tmpList = new LinkedList<>();
        List<String> resultList = new LinkedList<>();
        String tmpResult = new String();
        for (String str : words){
            if(str.trim().length() > 0){
                if((str + tmpResult + " ").length() < maxWidth){
                    str += " ";
                    tmpResult += str;
                } else {
                    tmpList.add(tmpResult);
                }
            }
        }
        if(tmpList.size() > 0){
            for(String str : tmpList){
                String[] tmpStr = str.split(" ");
                String tmpLine = new String();
                String line = new String();
                for (String word : tmpStr){
                    tmpLine += word;
                }
                if(tmpList.size() > 1){
                    int remainer = (maxWidth - tmpLine.length()) % (tmpStr.length - 1);
                    int ava = (maxWidth - tmpLine.length()) / (tmpStr.length - 1);
                    for(int i = 0; i < tmpStr.length; i++){
                        String word = tmpStr[i];
                        while(ava > 0){
                            word += " ";
                            ava--;
                        }
                        if(i < remainer){
                            word += " ";
                        }
                        line += word;
                    }
                } else {
                    int remainer = maxWidth - tmpLine.length();
                    line = tmpStr[0];
                    while(remainer-- > -1){
                        line += " ";
                    }
                }

                resultList.add(line);
            }
        } else {
            resultList.add("");
        }

        return resultList;
    }
}
```
