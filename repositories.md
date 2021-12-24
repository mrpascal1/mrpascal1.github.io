---
layout: default
---


<style>
	.repository-block {
		border: 1px solid red;
		margin-bottom: 1em;
		margin-top: 1em;
		padding: 1em;
	}
</style>

<pre>{{ site.github | jsonify }}</pre>

{% assign repos = site.github.public_repositories %}

{% for repo in repos %}

<div class="repository-block"  data-id="{{ repo.id }}">
	<h3>{{ repo.name }}</h3>
	<p>{{ repo.description }}</p>
	<p>{{ repo.topics }}</p>
	{% highlight json %}{{ repo }}{% endhighlight %}
</div>

{% endfor %}