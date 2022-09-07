---
layout: post
title:  Using Ruby to Translate Roman Numerals to Integers
date:   2022-09-06 16:11:57.474405748 -0400
image:  '/images/roman-numerals.jpg'
tags:   [ruby, leetcode, strings, arrays, hashes]
read_time: 3
comments: true
---
Photo by <a href="https://unsplash.com/@climatechangevi?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Karl Callwood</a> on <a href="https://unsplash.com/s/photos/roman-numeral?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  
## Introduction

Recently, I have been trying to maintain my coding chops by regularly submitting leetcode and codewars solutions. I thought I'd start with the 'easy' ones first to warm up. This one was categorized as easy but only has a 58% acceptance rate. 

Here is a link to the problem on [leetcode](https://leetcode.com/problems/roman-to-integer/) with instructions.

### Defining the problem
Basically, the goal is to write a function, that, given a Roman numeral string as an input, outputs the equivalent integer, e.g., `"MCMXCIV" => 1994`.

### Thinking about data structures.

My first thought was that I should map the fundamental Roman numerals to their equivalent values using a hash map, like so:

{% highlight ruby %}
def numerals
  {
    'I' => 1,
    'V' => 5,
    'X' => 10,
    'L' => 50,
    'C' => 100,
    'D' => 500,
    'M' => 1000
  }
end
{% endhighlight %}

I noted I am also dealing with a string and likely an array.

### Manipulating strings and arrays

I realized I will also need to evaluate the characters of the string. Ruby has a handy `string#each_char` method that iterates over each individual character of a string, so `'VIII'` would be split into an array of characters as `['V', 'I', 'I', 'I' ]`. Now that I am working with an array, I have access to the `array#map` method, which can be used to map each numeral character to its value:

{% highlight ruby %}

['V', 'I', 'I', 'I'].map { |n| numerals[n] }
# => [5, 1, 1, 1]

# using built-in array#sum method
['V', 'I', 'I', 'I'].map { |n| numerals[n] }.sum

# => [5, 1, 1, 1].sum
# => 8

{% endhighlight %}

So we're done right? We've basically covered the process we already do in our head (5 + 1 + 1 + 1 = 8)

### Gotchas

However, the tricky part comes when Roman numerals signify a number that is one factor less than a multiple of 5 or 10, e.g. IV is one less than V and means subtract one from five and return four. From the exercise instructions linked above:

- I can be placed before V (5) and X (10) to make 4 and 9. 
- X can be placed before L (50) and C (100) to make 40 and 90. 
- C can be placed before D (500) and M (1000) to make 400 and 900.

This throws a wrench in our plans to simply sum the mapped characters of the string, because the above method does not "group" the numerals in anyway. Perhaps there is a way to extract those special cases, e.g. IX and CM.

First, let's also define them in a hash like the one above:

{% highlight ruby %}
def special_numerals
  {
    'IV' => 4,
    'IX' => 9,
    'XL' => 40,
    'XC' => 90,
    'CD' => 400,
    'CM' => 900
  }
end
{% endhighlight %}

### The scan method

Now we need a way to scan the string and check for these special numerals. Thankfully, Ruby makes it easy with the `string#scan` method which returns an array of matches found in the string:

{% highlight ruby %}
  s = "MCMXCIV" # 1994
  special_matches = s.scan(/IV|IX|XL|XC|CD|CM/)
  # => ["CM", "XC", "IV"]
{% endhighlight %}

This allows us to use the same "map and sum" logic used above. But what to do with the remaining "non-special" characters. This will potentially result in counting some numerals twice. This can be prevented with a simple check.

First, check if there are any special numerals used in the string. If so, map them to their values and sum them. Record this sum in a variable.

### The gsub method

Second, remove those "special" numerals from the string so that the remaining numerals can be summed accurately. This is where the handy `string#gsub` method comes in:

{% highlight ruby %}
  s = "MCMXCIV" # 1994
  special_matches = s.scan(/IV|IX|XL|XC|CD|CM/)
  # => ["CM", "XC", "IV"]
  special_sum = special_matches.map { |m| special_numerals[m]}.sum
  # => 900 + 90 + 4 == 994
  special_matches.each { |m| s.gsub!(m, '') }
  # s is modified in place with its special numerals removed.
  # => s == "M"
  # we still need to add 1000 to our result
{% endhighlight %}

And so, the input string is now "M" and can be summed using the same logic. In this case, there is only one character, but the logic is the same for multiple characters:

{% highlight ruby %}
  normal_sum = s.each_char.map { |c| numerals[c] }.sum  
  # => numerals['M'] == 1000
  # => [1000].sum
  # => 1000
{% endhighlight %}

### Solution

Putting it all together we return the "special" numeral sum plus the "normal" numeral sum if there are any "special" numerals in the string. Otherwise, we return the "normal" sum. In code, I translated this logic to the following solution:

{% highlight ruby %}
def roman_to_int(s)
  special_matches = s.scan(/IV|IX|XL|XC|CD|CM/)
  if special_matches
    special_sum = special_matches.map { |m| special_numerals[m]}.sum
    special_matches.each { |m| s.gsub!(m, '') }
  end
  normal_sum = s.each_char.map { |c| numerals[c] }.sum  
  return (special_sum + normal_sum) if special_sum
  
  normal_sum
end

def numerals
  {
    'I' => 1,
    'V' => 5,
    'X' => 10,
    'L' => 50,
    'C' => 100,
    'D' => 500,
    'M' => 1000
  }
end

def special_numerals
  {
    'IV' => 4,
    'IX' => 9,
    'XL' => 40,
    'XC' => 90,
    'CD' => 400,
    'CM' => 900
  }
end

roman_to_s("MCMXCIV")
# => 1994
{% endhighlight %}

## Conclusion

I do not think this is the fastest solution on leetcode, but I personally find it readable. This is a testament to the power of the hash data structure, which made it straighforward to map a numeral to its integer value. 

How did you solve this problem or what could I do differently? Let me know in the comments below!