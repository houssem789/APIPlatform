# APIPlatform

Les relations entre entités, il y a trois: OneToOne, ManyToOne et ManyToMany.

Dans notre exemple, la relation entre Article et Tag est de type ManyToMany, c'est à dire qu'un objet Article peut être lié à 0 ou plusieurs objets Tag et inversement. La relation entre Article et Comment est de type ManyToOne, ce qui veut qu'un objet Article peut avoir 0 ou plusieurs objets Comment, tandis qu'un objet Comment ne peut être lié qu'à un seul objet Article.

Nous allons commencer par mettre en place la relation entre les entités Article et Comment. Dans une relation ManyToOne, c'est le coté Many qui doit définir la relation, ici c'est l'entité Comment qui est le coté Many, c'est donc dans cette entité que nous allons définir la relation. Nous allons le faire à traver la ligne de commande, ouvre donc un terminal et place toi à la racine du projet puis exécute la commande:

\$ php bin/console make:entity
BashCopy
La console va ensuite te demander plusieurs questions:

d'abord l'entité à créer ou modifier, dans notre cas nous allons modifier l'entité Comment
puis la nouvelle propriété que nous voulons ajouter, je l'ai appeler article
ensuite le type du champ, nous allons créer une relation de type ManyToOne, il faut donc saisir ManyToOne
l'entité à laquelle cette relation sera rattaché, dans notre cas c'est l'entité Article
ensuite pouvons-nous créer un commentaire sans article? Non, la réponse c'est donc no
voulons-nous avoir une relation bidirectionnelle, c'est a dire récupérer les commentaires d'un article depuis l'entité Article, on répond yes
le nom du champ à ajouter dans l'entité Article, comments, tu fais juste Entrer et c'est bon
la dernière question demande si on doit supprimer un commentaire à chaque fois que l'on supprime un article, yes
Voilà, tu peux regarder cette image pour plus de compréhension:

A ce point, si tu regardes l'entité Comment, un nouvel attribut \$article a été ajouté:

<?php
// src/Entity/Comment.php

namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(
 *     normalizationContext={"groups"={"comment:read"}},
 *     denormalizationContext={"groups"={"comment:write"}}
 * )
 * @ORM\Entity(repositoryClass="App\Repository\CommentRepository")
 */
class Comment
{
    // ...

    /**
     * @ORM\ManyToOne(targetEntity="App\Entity\Article", inversedBy="comments")
     * @ORM\JoinColumn(nullable=false)
     */
    private $article;

    // ...

    public function getArticle(): ?Article
    {
        return $this->article;
    }

    public function setArticle(?Article $article): self
    {
        $this->article = $article;

        return $this;
    }
}
PHPCopy
Et dans l'entité Article aussi, un nouvel attribut $comments a été ajouté:

<?php
// src/Entity/Article.php

namespace App\Entity;

use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;
use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(
 *     normalizationContext={"groups"={"article:read"}},
 *     denormalizationContext={"groups"={"article:write"}}
 * )
 * @ORM\Entity(repositoryClass="App\Repository\ArticleRepository")
 */
class Article
{
    // ...

    /**
     * @ORM\OneToMany(targetEntity="App\Entity\Comment", mappedBy="article", orphanRemoval=true)
     */
    private $comments;

    // ...

    /**
     * @return Collection|Comment[]
     */
    public function getComments(): Collection
    {
        return $this->comments;
    }

    public function addComment(Comment $comment): self
    {
        if (!$this->comments->contains($comment)) {
            $this->comments[] = $comment;
            $comment->setArticle($this);
        }

        return $this;
    }

    public function removeComment(Comment $comment): self
    {
        if ($this->comments->contains($comment)) {
            $this->comments->removeElement($comment);
            // set the owning side to null (unless already changed)
            if ($comment->getArticle() === $this) {
                $comment->setArticle(null);
            }
        }

        return $this;
    }
}
PHPCopy
Il faut maintenant ajouter le groupe article:read à l'attribut $comments dans Article.php. Ce que nous voulons c'est qu'à chaque appel des routes GET /api/articles et GET /api/articles/{id}, nous retournons chaque article avec sa liste de commentaires:

<?php
// src/Entity/Article.php

namespace App\Entity;

// ...
class Article
{
    // ...

    /**
     * @ORM\OneToMany(targetEntity="App\Entity\Comment", mappedBy="article", orphanRemoval=true)
     *
     * @Groups("article:read")
     */
    private $comments;

    // ...
}
PHPCopy
Si tu regardes maintenant le schéma de l'entité Article sur Swagger, nous avons bien un tableau comments, qui est encore vide pour l'instant:



Nous avons ajouter le groupe article:read à $comments, mais nous n'avons pas spécifier quels attributs de l'entité Comment on veut retourner en sérialisant l'objet Article, il faut donc aller dans la classe Comment.php et ajouter là aussi le groupe article:read aux attributs que nous voulons retourner en sérialisant Article:

<?php
// src/Entity/Comment.php

namespace App\Entity;

// ...
class Comment
{
    /**
     * @ORM\Id()
     * @ORM\GeneratedValue()
     * @ORM\Column(type="integer")
     *
     * @Groups({"comment:read", "article:read"})
     */
    private $id;

    /**
     * @ORM\Column(type="text")
     *
     * @Groups({"comment:read", "comment:write", "article:read"})
     */
    private $content;

    /**
     * @ORM\Column(type="datetime")
     *
     * @Groups({"comment:read", "article:read"})
     */
    private $createdAt;

    /**
     * @ORM\ManyToOne(targetEntity="App\Entity\Article", inversedBy="comments")
     * @ORM\JoinColumn(nullable=false)
     */
    private $article;

    // ...
}
PHPCopy
J'ai choisi d'ajouter le groupe article:read sur tous les trois attributs de Comment, mais cela va dépendre de vous, donc vous allez ajouter le groupe qu'aux attributs que vous voulez retourner avec l'article. Si tu actualises la page, tu remarques tout de suite le changement sur les schémas:



Il faut maintenant mettre à jour la base de données avec:

$ php bin/console make:migration
$ php bin/console doctrine:migrations:migrate
BashCopy
Si tu exécutes la route GET /api/articles tu verras que chaque article contient un tableau comments[], vide pour l'instant.



Pour ajouter un commentaire, il faut mentionner le contenu du commentaire, l'article auquel est rattaché le commentaire et l'auteur du commentaire dont nous parlerons plus tard. Pour l'instant nous allons juste gérer le contenu et l'article.

Si tu regardes le body de la requête pour la route POST /api/comments, tu remarqueras que nous avons juste l'attribut content:



Et si nous essayons d'ajouter un commentaire, nous avons une erreur qui dit que la colonne article_id ne peut pas être null, et c'est nous même qui l'avons défini lors de la création de la relation, un commentaire doit toujours être relié à un article. Pour corriger cela, nous allons modifier la classe Comment et ajouter le groupe comment:write à l'attribut article:

<?php
// src/Entity/Comment.php

namespace App\Entity;

// ...
class Comment
{
    // ...

    /**
     * @ORM\ManyToOne(targetEntity="App\Entity\Article", inversedBy="comments")
     * @ORM\JoinColumn(nullable=false)
     *
     * @Groups("comment:write")
     */
    private $article;

    // ...
}
PHPCopy
Et le schéma du body pour la route POST /api/comments est modifier et prend maintenant le contenu et l'article:

{
  "content": "string",
  "article": "string"
}
JSONCopy
Il y a maintenant un paramètre article de type string. Alors tu te dis que normalement le paramètre article doit plutôt être de type Article et je suis d'accord avec toi. Mais dis toi qu'ici, nous créons un nouveau commentaire que nous allons rajouter a un article déjà existant, nous ne créons pas un nouvel article. Nous allons donc envoyer l'identifiant unique de l'article.

Si on avait fait l'inverse, c'est à dire si on avait ajouté le groupe article:write à l'attribut comments dans Article.php, on aurait fait de sorte qu'à la création d'un article, que l'on puisse directement l'ajouter un ou plusieurs commentaires, mais cela ne doit pas être notre cas. Nous verrons un exemple dans la relation entre Article et Tag.

Tout à l'heure j'ai parler de l'identifiant unique d'un article, que nous allons envoyer a la création d'un commentaire. Alors cet identifiant est différent de l'attribut id que nous retrouvons dans toutes nos entités.

Si tu navigues sur la route http://127.0.0.1:8000/api/articles.jsonld tu arrives sur une page comme sur cette image:



Chaque article contient une clé @id qui est différent de l'attribut id, cette clé @id représente l'identifiant unique de chaque ressource sur toute notre application, elle est composée du chemin de base pour la collection d'articles /api/articles et de l'attribut id de l'article. C'est donc cette clé @id qui est l'identifiant unique de notre ressource et c'est cette chaîne que nous enverrons dans le body de la requête pour la création d'un commentaire.

Nous allons créer un nouveau commentaire et l'ajouter à l'article avec l'id 2, le body sera donc ceci:

{
  "content": "Superbe article! Merci beaucoup.",
  "article": "api/articles/2"
}
JSONCopy
Et le commentaire a bien été ajouter.

Et si je récupère l'article 2 http://127.0.0.1:8000/api/articles/2, j'ai bien le commentaire que j'ai ajouter dans le tableau comments:



Maintenant que nous avons la relation entre Article et Comment, il nous faut la relation entre Article et Tag, c'est pratiquement la même chose sauf qu'ici la relation est de type ManyToMany. Je définirai l'entité Article comme étant l'entité propriétaire. Nous allons donc créer la relation avec make:entity:



Je modifie donc l'entité Article pour lui rajouter un attribut tags avec "s" et je demande à ce qu'un autre attribut articles soit ajoutés dans la classe Tag.php.

Il faut ensuite mettre à jour la base de données avec:

$ php bin/console make:migration
$ php bin/console doctrine:migrations:migrate
BashCopy
Nous allons ajouter les groupes article:read et article:write à l'attribut tags dans la classe Article.php et à l'attribut label dans la classe Tag.php:

<?php
// src/Entity/Article.php

namespace App\Entity;

// ...
class Article
{
    // ...

    /**
     * @ORM\ManyToMany(targetEntity="App\Entity\Tag", inversedBy="articles")
     *
     * @Groups({"article:read", "article:write"})
     */
    private $tags;

    // ...
}
PHPCopy
<?php
// src/Entity/Tag.php

namespace App\Entity;

// ...
class Tag
{
    // ...

    /**
     * @ORM\Column(type="string", length=255)
     *
     * @Groups({"tag:read", "tag:write", "article:read", "article:write"})
     */
    private $label;

    // ...
}
PHPCopy
Le nouveau schéma pour les méthodes POST, PUT et PATCH pour la ressource /api/articles va donc ressembler à ceci:

{
  "title": "string",
  "content": "string",
  "picture": "string",
  "tags": [
    {
      "label": "string"
    }
  ]
}
JSONCopy
Nous allons donc envoyer dans le body de la requête, le title, content, picture et un tableau d'objet Tag qui contient le label. Nous allons tout de suite essayer cela avec ce body:

{
  "title": "Apprendre a faire un blog avec Symfony 5",
  "content": "",
  "picture": "",
  "tags": [
    {
      "label": "php"
    },
    {
      "label": "symfony"
    },
    {
      "label": "composer"
    }
  ]
}
JSONCopy
Nous avons une erreur qui dit que nous avons des entités qui ne sont pas persistés. Nous essayons de créer un nouvel article, et en même temps nous allons créer trois nouveaux tags, il faut donc spécifier à doctrine de persister les tags au même moment que l'article, pour cela, il faut juste rajouter la propriété cascade="persist" dans la relation entre Article et Tag dans la classe Article comme ceci:

<?php
// src/Entity/Article.php

namespace App\Entity;

// ...
class Article
{
    // ...

    /**
     * @ORM\ManyToMany(targetEntity="App\Entity\Tag", inversedBy="articles", cascade="persist")
     *
     * @Groups({"article:read", "article:write"})
     */
    private $tags;

    // ...
}
PHPCopy
Et si on réessaye cette fois, ça passe sans problème.



Mais nous risquons d'avoir un problème avec ce modèle, d'abord, il faut s'assurer qu'un tag est unique en base de données, donc que le label php ne soit pas dupliqué, pour cela il faut modifier le fichier Tag.php, dans l'annotation, ajouter la propriété unique=true à l'attribut label:

<?php
// src/Entity/Tag.php

namespace App\Entity;

// ...
class Tag
{
    /**
     * @ORM\Column(type="string", length=255, unique=true)
     *
     * @Groups({"tag:read", "tag:write", "article:read", "article:write"})
     */
    private $label;

    // ...
}
PHPCopy
Si tu essaies maintenant de créer un nouvel article avec un tag php ou symfony ou composer, tu auras un problème qui dit qu'il y a un doublon.

Il faut savoir que ce qui se passe actuellement, c'est qu'à chaque fois que tu essaies de créer un nouvel article, une liste de nouveau tag est créé en fonction des paramètres que tu envoies., et cela n'est pas bon. Ce que nous allons faire, c'est de vérifier si le tag existe, on ne le crée pas, donc on le persiste pas, on l'ajoute juste à l'article. S'il existe pas, alors dans ce cas on peut le persister. Pour cela, nous allons modifier le data persister qu'on avait déjà créer, ArticleDataPersister.php, je vais te mettre son contenu en intégral:

<?php
// src/DataPersister/ArticleDataPersister.php

namespace App\DataPersister;

use App\Entity\Tag;
use App\Entity\Article;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\String\Slugger\SluggerInterface;
use ApiPlatform\Core\DataPersister\ContextAwareDataPersisterInterface;

/**
 *
 */
class ArticleDataPersister implements ContextAwareDataPersisterInterface
{
    /**
     * @var EntityManagerInterface
     */
    private $_entityManager;

    /**
     * @param SluggerInterface
     */
    private $_slugger;

    /**
     * @param Request
     */
    private $_request;

    public function __construct(
        EntityManagerInterface $entityManager,
        SluggerInterface $slugger,
        RequestStack $request
    ) {
        $this->_entityManager = $entityManager;
        $this->_slugger = $slugger;
        $this->_request = $request->getCurrentRequest();
    }


    /**
     * {@inheritdoc}
     */
    public function supports($data, array $context = []): bool
    {
        return $data instanceof Article;
    }

    /**
     * @param Article $data
     */
    public function persist($data, array $context = [])
    {
        // Update the slug only if the article isn't published
        if (!$data->getIsPublished()) {
            $data->setSlug(
                $this
                    ->_slugger
                    ->slug(strtolower($data->getTitle())). '-' .uniqid()
            );
        }

        // Set the updatedAt value if it's not a POST request
        if ($this->_request->getMethod() !== 'POST') {
            $data->setUpdatedAt(new \DateTime());
        }

        $tagRepository = $this->_entityManager->getRepository(Tag::class);
        foreach ($data->getTags() as $tag) {
            $t = $tagRepository->findOneByLabel($tag->getLabel());

            // if the tag exists, don't persist it
            if ($t !== null) {
                $data->removeTag($tag);
                $data->addTag($t);
            } else {
                $this->_entityManager->persist($tag);
            }
        }

        $this->_entityManager->persist($data);
        $this->_entityManager->flush();
    }

    /**
     * {@inheritdoc}
     */
    public function remove($data, array $context = [])
    {
        $this->_entityManager->remove($data);
        $this->_entityManager->flush();
    }
}
PHPCopy
Les sous ressources
Nous allons maintenant parler des sous ressources. Un exemple de sous ressources c'est la liste des commentaires d'un article. Pour l'instant, nous récupérons la liste des commentaires d'un article directement en récupérant un article, ce qui est pratique quand nous n'avons pas beaucoup de données à retourner. Mais imagine que nous avons 100 articles qui ont chacun 500 commentaires, ça fait beaucoup de données et cela peut impacter la performance de notre application. Bon tu vas me dire que nous avons la pagination, mais nous allons retourner combien de ressources par page, disons 20, on aura donc 20 articles qui auront chacun 20 commentaires, on est toujours pas tiré d'affaire. L'idéal serait d'avoir une route /api/articles/{id}/comments pour retourner tout les commentaires de l'article id, top n'est ce pas? La même chose serait idéal pour les tags avec /api/tags/{id}/articles pour avoir tous les articles d'un tag.

Dans le premier cas (/api/articles/{id}/comments), comments est une sous ressource et dans le second cas (/api/tags/{id}/articles), articles est aussi une sous ressource.

Pour définir une sous ressource, il suffit d'ajouter l'annotation @ApiSubresource à l'attribut que tu veux définir comme une sous ressource.

Pour /api/articles/{id}/comments, il faut modifier la classe Article.php et ajouter l'annotation @ApiSubresource à l'attribut comments:

<?php
// src/Entity/Article.php

namespace App\Entity;

// ...
use ApiPlatform\Core\Annotation\ApiSubresource;

// ...
class Article
{
    // ...

    /**
     * @ORM\OneToMany(targetEntity="App\Entity\Comment", mappedBy="article", orphanRemoval=true)
     * @ApiSubresource
     *
     * @Groups("article:read")
     */
    private $comments;

    // ...
}
PHPCopy
Et pour la route /api/tags/{id}/articles, il faut modifier la classe Tag.php et ajouter l'annotation @ApiSubresource a l'attribut articles.

Et voilà, nous avons la relation entre nos entités!
