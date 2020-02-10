The HTML5 Speech Synthesis API isn't widely supported, but it is something cool to play with and can be useful. It is supported only in Chrome and Safari. For a recent prototype feature I used it. It is definitely still pretty beta. The goal of this post is to give some pointers when using it, if you choose to.

## Detecting Support

If you are going to do anything with Speech Synthesis you are probably going to want to enable it only if your user's browser supports it.

{% highlight javascript %}
supportsSpeechSynthesis = function (){
  return 'speechSynthesis' in window
}
supportsSpeechSynthesis() // true or false
{% endhighlight %}

## Basic Usage

Using the API is actually really simple. Open your javascript console and try it out.

{% highlight javascript %}
phrase = "The quick brown fox jumps over the lazy dog."
audio = new SpeechSynthesisUtterance(phrase)
window.speechSynthesis.speak(audio)
{% endhighlight %}

## Chrome Bugs

There is a [known bug in Chrome](https://code.google.com/p/chromium/issues/detail?id=335907) that causes the speech synthesis to stop working if you use text greater than ~ 300 characters. The only way to get to start working again is to restart Chrome. The simple solution is to split up your text into sentences. This actually works out for the better, since there will be a larger gap between the sentences and it sounds more natural.

{% highlight javascript %}
phrase = "\
Oak is strong and also gives shade.\
Cats and dogs each hate the other.\
The pipe began to rust while new.\
Open the crate but don't break the glass.\
Add the sum to the product of these three.\
Thieves who rob friends deserve jail.\
The ripe taste of cheese improves with age.\
Act on these orders with great speed.\
The hog crawled under the high fence.\
Move the vat over the hot fire.\
"

sentences = phrase.split(".")
for (i = 0; i < sentences.length; i++) {
  sentence = sentences[i]
  audio = new SpeechSynthesisUtterance(sentence)
  window.speechSynthesis.speak(audio)
}
{% endhighlight %}

This works well, however as a safeguard you might want to make sure none of your sentences go over the limit. If any do break them up!

## Pausing and Resuming

The API supports stopping, pausing, and resuming. Safari will resume at the beginning of the current SpeechSynthesisUtterance object whereas chrome will resume where it left off.

{% highlight javascript %}
window.speechSynthesis.pause()
window.speechSynthesis.resume()
window.speechSynthesis.stop()
{% endhighlight %}

I've found that to get everything to work consistently sometimes you have to pause and resume it once or twice at the beginning of playback.

## Related Links

*   [HTML5 Rocks Blog Post](http://updates.html5rocks.com/2014/01/Web-apps-that-talk---Introduction-to-the-Speech-Synthesis-API)
*   [Treehouse Blog Post on Speech Synthesis API](http://blog.teamtreehouse.com/getting-started-speech-synthesis-api)
*   [Can I Use](http://caniuse.com/#feat=speech-synthesis)
*   [Harvard Sentences](http://en.wikipedia.org/wiki/Harvard_sentences)
*   [Google Chrome Issue](https://code.google.com/p/chromium/issues/detail?id=335907)
