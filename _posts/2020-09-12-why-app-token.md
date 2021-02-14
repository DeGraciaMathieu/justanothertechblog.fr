---
layout: post
date: 2021-02-14 12:36
title: "Pourquoi le hash de password est hardcodé dans UserFactory ?"
description: Pourquoi le hash de password est hardcodé dans UserFactory ?
comments: false
category: 
- laravel
tags:
- laravel
- exploration
---

<figure class="aligncenter">
    <img src="https://images.unsplash.com/photo-1592483335937-a3213ac4a833?ixid=MXwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHw%3D&ixlib=rb-1.2.1&auto=format&fit=crop&w=3900&q=80" />
</figure>

Lors d'une soirée live coding sur la chaine twitch de [DevMaker_tv](https://www.twitch.tv/devmaker_tv) nous nous sommes intéressés à une subtilité de Laravel présente dans le fichier `UserFactory`.

{% highlight php linenos %}
public function definition()
{
    return [
        'name' => $this->faker->name,
        'email' => $this->faker->unique()->safeEmail,
        'email_verified_at' => now(),
        'password' => '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', // password
        'remember_token' => Str::random(10),
    ];
}
{% endhighlight %}

Comment est-ce possible que cet hash de password soit présent en dur dans notre `UserFactory` alors que la valeur de `APP_TOKEN` de l'application n'est pas encore définie ?

J'ai toujours bêtement consideré que `APP_TOKEN` était utilisé comme salt pour tous les hash, pour autant lors d'une nouvelle installation de Laravel la valeur de `APP_TOKEN`... est null.

Creusons dans le code pour déterminer le fin mot de l'histoire.

## TLDR

Le `APP_TOKEN` est uniquement utilisé pour [l'encryption](https://laravel.com/docs/8.x/encryption).

Un password en base de données est quant à lui hashé par un [password_hash](https://www.php.net/manual/fr/function.password-hash.php) avec par défaut un `bcrypt`.

Cet hash de password est présent en dur pour des soucis d'optimisation, un `password_hash` étant fortement demandeur en ressources.

## Exploration 

Premières constatations, il n'y a rien de natif dans Laravel pour hash automatiquement la valeur d'un password lors de l'insertion d'un utilisateur.

C’est aux développeurs de gérer ce besoin… et la documentation n’est pas forcément très loquace sur la marche à suivre.

Seule véritable information, on apprend que la méthode `Auth::attempt()`, permettant d'authentifier un utilisateur, procedera à un hash du password pour le comparer avec celui présent en base de données.

> The attempt method accepts an array of key / value pairs as its first argument. The values in the array will be used to find the user in your database table. If the user is found, **the hashed password stored in the database will be compared with the password value passed** to the method via the array. 

Pour comprendre ce qui se passe, nous allons retracer le code utilisé lors d'une tentative d'authentification pour déterminer comment Laravel hash un password.

## Comprendre l'authentification

Une tentative d'authentification fonctionne deux temps : 

- Récupérer un utilisateur.
- Vérifier que le password envoyé est correct. 

Le workflow d'authentification de Laravel est complexe et totalement configurable, la récupération d'un utilisateur et l'objet que retournera l'authentification sont dissociés.

Par défaut, la récupération de l'utilisateur se trouve dans la class `EloquentUserProvider` et l'objet retourné est un model `User`.

Ce choix est configurable dans le fichier `config\auth.php`.

{% highlight php linenos %}
'providers' => [
    'users' => [
        'driver' => 'eloquent', // EloquentUserProvider 
        'model' => App\Models\User::class,
    ],
{% endhighlight %}

Lors d'une tentative d'authentification, apres avoir récupéré l'utilisateur à l'aide de la méthode `retrieveByCredentials` du `EloquentUserProvider`, c'est au rôle de la méthode `validateCredentials` de vérifier la pertinence du password à l'aide d'un objet `hasher`.  

{% highlight php linenos %}
public function validateCredentials(UserContract $user, array $credentials)
{
    $plain = $credentials['password'];

    return $this->hasher->check($plain, $user->getAuthPassword());
}
{% endhighlight %}

Après vérification, il s'avère que cette variable `hasher` est une instance de `BcryptHasher`, que Laravel génère depuis un design pattern [Manager](https://github.com/DeGraciaMathieu/Manager).

C'est ce `BcryptHasher` qui aura la charge de vérifier le pertinence du password.

## Le check du password

Nous y sommes, voici comment Laravel hash un password lors d'une authentification. 

{% highlight php linenos %}
public function check($value, $hashedValue, array $options = [])
{
    if (strlen($hashedValue) === 0) {
        return false;
    }

    return password_verify($value, $hashedValue);
}
{% endhighlight %}

On s'aperçoit que Laravel utilise une fonction [password_verify](https://www.php.net/manual/fr/function.password-verify.php) native à php... et aucune présence du `APP_TOKEN` en tant que salt. 

On peut donc affirmer que la valeur du `APP_TOKEN` n'impacte pas le hash des passwords.

Alors à quoi sert `APP_TOKEN` ?

## Hasher n'est pas crypter

Le `APP_TOKEN` est finalement utilisé en tant que salt pour crypter des informations depuis la class [Crypt](https://laravel.com/docs/8.x/encryption).

Les passwords des utilisateurs sont quant à eux hashé à l'aide d'un algorithme que vous pouvez configuer dans `Config\hashing.php` :

C'est pour cette raison que `UserFactory` contient un hash en dur, `$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi` correspondra toujours à la valeur `password`.

Bonne journée.
