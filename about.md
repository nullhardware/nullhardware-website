---
sidebar: |
    ## We're the approachable open-source hackers.
    {: .h4}
    
    *Our mission is to help makers and hackers master embedded development.*

    We have a combined 30+ years of industry experience in firmware and circuit design, and the marjority of us have graduate degrees. We are currently accepting a *limited* number of new clients.

    Looking for experienced embedded developers? Contact [consult@nullhardware.com](mailto:consult@nullhardware.com).
    {: .alert .alert-danger }

    ---
---

# Are you a maker or hacker?

Do you love **Arduinos**, **LEDs**, **Raspberry Pis**, **Microcontrollers**, and **Breadboards** *but* are stuck doing *blinky lights* when what you really want is *internet-connected laser-powered drones*?
{: .lead }

## *We can help get you there.*  
  
>
> Our team met at University over 8 years ago while studying Electrical Engineering. We've watched first-hand the explosive growth of the maker movement driven by platforms like the Arduino, Raspberry Pi, and ESP32. But trying to get useful development information for all of those platforms is time consuming, poorly organized, and fraught with bad advice. 
> 
> Sometimes what you *really* need is access to organized, accurate, and concise information. Data sheets are great, but a summary of 10 different pages from 10 different app notes might be what you really need. Starting in May 2017, we're committed to helping out makers and hackers by documenting, sharing, and developing open hardware and software. 
> 
> *We weren't always this way*. As hackers, we started out trying to tear things apart. We reverse-engineered oscilloscope modules and tried cracking BIOS passwords with BusPirates. These may have been fun projects, but they didn't help anyone just starting out. So we've changed our name and our mission to reflect our new vision - a future where technology is accessible, programmable, and has unlocked potential.
> 

Open Hardware is successful when people know about it, learn about it, and actually use it. If you are as passionate about electronics and programming as we are, **share this site** with your friends, comment on our [blog](/blog/) posts, and send us your suggestions for topics to cover in our [Reference](/reference/) section.

{% include snips/share.html class="h1 text-center list-inline" url="/" %}

## Contributors

{% for authors in site.data.authors %}
  {% assign author = authors | first %}
  {% include snips/author.html author=author %}
{% endfor %}