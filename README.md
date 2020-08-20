# APIPlatform

Les relations entre entités, il y a trois: OneToOne, ManyToOne et ManyToMany.

Dans notre exemple, la relation entre Article et Tag est de type ManyToMany, c'est à dire qu'un objet Article peut être lié à 0 ou plusieurs objets Tag et inversement. La relation entre Article et Comment est de type ManyToOne, ce qui veut qu'un objet Article peut avoir 0 ou plusieurs objets Comment, tandis qu'un objet Comment ne peut être lié qu'à un seul objet Article.

Si on avait fait l'inverse, c'est à dire si on avait ajouté le groupe article:write à l'attribut comments dans Article.php, on aurait fait de sorte qu'à la création d'un article, que l'on puisse directement l'ajouter un ou plusieurs commentaires, mais cela ne doit pas être notre cas. Nous verrons un exemple dans la relation entre Article et Tag.

Maintenant que nous avons la relation entre Article et Comment, il nous faut la relation entre Article et Tag, c'est pratiquement la même chose sauf qu'ici la relation est de type ManyToMany.
