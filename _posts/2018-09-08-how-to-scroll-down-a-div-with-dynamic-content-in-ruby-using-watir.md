---
title: How to scroll down a div with dynamic content in Ruby using Watir
tags: [ruby, watir, dynamic-content]
categories: [TIL, Ruby]
---

Explaining how to scroll down a div with dynamic content as long as there is more content to show and also some stuff like `scrollHeight`, `scrollTop` and `clientHeight`.

[Take me right there!][3]

---
I was trying to create a simple Instagram bot in `Ruby` using [Watir gem][1]. There are of course lots of open source Instagram bots already in GitHub. But I wanted to see if I could write something similar which I thought would be fun.
 
One thing I wanted to figure out was how to unfollow non-followers. This feature is a little bit tedious, because, since I am not using any Instagram APIs, if I wanted to unfollow non-followers (I love to say this!) I would need to know the usernames of the people following me and the ones I am following. Then all I need to do is just compare the two arrays (the ones following me and the ones I am following) and extract the ones I am following and also who are not following me into another array (I know this got really confusing). Then just unfollow them, right?

<figure>
	<a href="{{ site.url}}/images/2018-09-08-how-to-scroll-down-a-div-with-dynamic-content-in-ruby-using-watir/followers_preview.gif" class="image-popup"><img src="{{ site.url}}/images/2018-09-08-how-to-scroll-down-a-div-with-dynamic-content-in-ruby-using-watir/followers_preview.gif" alt="Instagram followers dialog box"></a>
</figure>

To do the above method using Watir gem in Ruby, I just first navigate to user's Instagram profile, then click on followers. But Instagram doesn't load all of your followers at once. You need to keep scrolling down as long as there are more followers to show as in the gif above. So you can do the math of the waiting time if you had around even 1k followers which is exactly why I said "tedious".

So I was trying to implement something that keeps scrolling down until you hit the bottom eventually then get all the followers (or your followings) into an array.

My first attempt was to get the follower count from the profile page and then approximate how many times I would need to scroll down (`control + end`) the followers dialog window page.

```ruby
# Navigate to user's page  
  
browser.goto "https://www.instagram.com/#{username}"  
  
# Get the follower count  
  
follower_count = browser.a(href: "/#{username}/followers/").text.to_i  
  
# Click the followers button  

sleep(3)  
browser.a(href: "/#{username}/followers/").click  
  
# Focus on the followers dialog window  
  
sleep(1)  
browser.div(class: 'j6cq2').focus  
  
# I figured instagram loads around 10 followers in each load. 
# Which is why I divided the follower_count by 10 and ceiled it.
    
scroll_count = (follower_count / 10.0).ceil  
  
scroll_count.times do  
  browser.send_keys :control, :end  
  sleep(0.7)   
end  

# What is left to do is just to get the followers list.  

browser.as.each do |follower|  
  followers << follower.title if follower.title != ""  
end
```

<figure>
	<a href="{{ site.url}}/images/2018-09-08-how-to-scroll-down-a-div-with-dynamic-content-in-ruby-using-watir/follower_count.gif" class="image-popup"><img src="{{ site.url}}/images/2018-09-08-how-to-scroll-down-a-div-with-dynamic-content-in-ruby-using-watir/follower_count.gif" alt="instagram follower count watir ruby"></a>
</figure>

---
<a name="tldr1"></a>
But as you see this is really time consuming. Another approach to this "scrolling down" stuff came [from a user][2] on the SO. I loved his approach and wanted to note it somewhere as to what exactly it does but I guess I've written a liiiittle bit too much. :)

So, here it is:
 
```ruby
scrollable = browser.div(class: 'j6cq2') # div with overflow-y=scroll
until browser.execute_script('return arguments[0].scrollTop + arguments[0].clientHeight >= arguments[0].scrollHeight', scrollable) do
    browser.execute_script('arguments[0].scrollTop = arguments[0].scrollHeight', scrollable)
    sleep(1)
end
```

#### What do all these mean?

`clientHeight` is actually the div's height which has scrollable content in it which is in this case the blue highlighted area's height in our case.


<figure>
	<a href="{{ site.url}}/images/2018-09-08-how-to-scroll-down-a-div-with-dynamic-content-in-ruby-using-watir/client_height.png" class="image-popup"><img src="{{ site.url}}/images/2018-09-08-how-to-scroll-down-a-div-with-dynamic-content-in-ruby-using-watir/client_height.png" alt="Client height"></a>
</figure>

---
`scrollTop` is the distance in pixels from the top of your content to it's top-most visible content. Let me explain this better for you. Take a look at the images below. 


<figure>
	<a href="{{ site.url}}/images/2018-09-08-how-to-scroll-down-a-div-with-dynamic-content-in-ruby-using-watir/scroll_top.png" class="image-popup"><img src="{{ site.url}}/images/2018-09-08-how-to-scroll-down-a-div-with-dynamic-content-in-ruby-using-watir/scroll_top.png" alt="scrollTop JavaScript"></a>
</figure>

I am 4 followers down from the top. I can't see those 4 followers and the top-most follower (which I can see) is the 5th one. You can also see this in the inspector. There are 4 `li` elements above and the 4th one is highlighted.

Since each`li` element's height is 54 pixels, our `scrollTop` would return 216 pixels which you can check like below in the picture.


---

`scrollHeight` is the total content height of the scrollable div.

<figure>
	<a href="{{ site.url}}/images/2018-09-08-how-to-scroll-down-a-div-with-dynamic-content-in-ruby-using-watir/scrollTop_clientHeight_scrollHeight.png" class="image-popup"><img src="{{ site.url}}/images/2018-09-08-how-to-scroll-down-a-div-with-dynamic-content-in-ruby-using-watir/scrollTop_clientHeight_scrollHeight.png" alt="scrollTop clientHeight scrollHeight"></a>
</figure>

Above picture shows how to find out all the three measurements and we had already calculated them. In this case I needed the class' name for the div with the scrollable content.

---

So you can now figure out how the above code works. Until `scrollTop + clientHeight = scrollHeight` it keeps assigning `scrollHeight` to `scrollTop` which causes page to scroll down.



[1]: http://watir.com/
[2]: https://stackoverflow.com/a/52227006/4796762
[3]: #tldr1
*[SO]: StackOverflow

