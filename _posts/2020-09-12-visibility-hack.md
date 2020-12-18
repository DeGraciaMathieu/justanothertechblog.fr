---
layout: post
date: 2020-09-12 12:36
title: "Acceder aux éléments protégés d'une class"
description: Basic information about the Steve.
comments: false
category: 
- php7
tags:
- ph7
- hack
- closure
- visibility
---

<figure class="aligncenter">
    <img src="https://images.unsplash.com/photo-1600456548090-7d1b3f0bbea5?ixlib=rb-1.2.1&ixid=MXwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHw%3D&auto=format&fit=crop&w=1350&q=80" />
</figure>

Nous sommes parfois amenés à vouloir **manipuler une propriété ou une méthode protected** de nos class.

L'utilisation de la [reflection](https://www.php.net/manual/fr/reflectionproperty.setaccessible.php) est envisageable pour modifier la visibilité mais une autre solution élégante à base de <code>closure</code> est également possible.

Considérons la class suivante :

{% highlight php linenos %}
class Foo {
    protected $bar;
    protected function bar() {}
}
{% endhighlight %}

L'idée est d'utiliser le `Closure::call` introduit [en php7](https://www.php.net/manual/fr/closure.call.php) pour **injecter une closure dans le scope d'une instance** de la class <code>Foo</code>.

{% highlight php linenos %}
$foo = new Foo();

$closure = function () {
    $this->bar = false;
    $this->bar();
};

$closure->call($foo);
{% endhighlight %}

Une fois notre closure injecté, il lui est désormais possible d'acceder à toutes les proprietés et méthodes protected de notre class.

Certes, c'est pas usuel, mais vous êtes majeur et vacciné. 