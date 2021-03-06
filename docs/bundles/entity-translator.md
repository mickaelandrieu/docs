Entity Translator
=================

Use and configure the EntityTranslator component in your Symfony project.

* [Component Documentation](http://elcodi.io/docs/components/entity-translator/)
* [Github Repository](https://github.com/elcodi/EntityTranslatorBundle)

For that, this implementation hides all the translation model behind a soft
layer, so your business model is never modified because of the translation
needs.

## About your model

Easy and simple. Let's work with a simple entity mapped in Doctrine.

``` php
/**
 * Class Product
 *
 * @ORM\Entity
 */
class Product
{
    /**
     * @var string
     *
     * @ORM\Id
     * @ORM\Column
     * @ORM\GeneratedValue
     *
     * id
     */
    protected $id;

    /**
     * @var string
     *
     * @ORM\Column(type="string")
     *
     * Name
     */
    protected $name;

    /**
     * @var string
     *
     * @ORM\Column(type="text")
     *
     * Description
     */
    protected $description;
}
```

This example will assume that you want to make both fields, name and
description, translatable. To make it happen, we can simply remove  the mapping
of both fields, to rely the responsibility of translation to the Translator
component.

``` php
/**
 * Class Product
 *
 * @ORM\Entity
 */
class Product
{
    /**
     * @var string
     *
     * @ORM\Id
     * @ORM\Column
     * @ORM\GeneratedValue
     *
     * id
     */
    protected $id;

    /**
     * @var string
     *
     * Name
     */
    protected $name;

    /**
     * @var string
     *
     * Description
     */
    protected $description;
}
```

Now your installation will see the Product as a simple id. Why is this good for
us? This example uses Annotations, and we highly recommend to define all your
mapping configuration using `yml` so you can easily decouple your model
implementation (entities) and your mapping configuration (Product.orm.yml) and
disable some fields for mapping with the same entity implementation.

## Configuring the entity translator

Now, we need to configure the translator. You need to define what entities and
fields you allow the engine to manage. Let's see the configuration for this
example.

``` yml
elcodi_entity_translator:
    configuration:
        My\Bundle\Entity\Product:
            alias: product

            // idGetter, by default, getId
            idGetter: getId
            fields:
                name:
                    // getter, by default, getName in this case
                    // setter, by default, setName in this case
                    getter: getName
                    setter: setName
                description:
                    getter: getDescription
                    setter: setDescription
```

This is the basic definition of the Product translation. You can see that there
is come specific information set by default, so indeed, this configuration could
be defined as well that way

``` yml
elcodi_entity_translator:
    configuration:
        My\Bundle\Entity\Product:
            alias: product
            fields:
                name: ~
                description: ~
```

You can use Interfaces as well, so in that case, to ensure that the
configuration will be still valid even if you change the Product implementation,
you can use an Interface for that

``` yml
elcodi_entity_translator:
    configuration:
        My\Bundle\Entity\Interfaces\ProductInterface:
            alias: product
            fields:
                name: ~
                description: ~
```

## Saving translations

Do you want to translate an entity? Great, let's do that! You can easily manage
your translation using the entity translator service.

``` php
$entityTranslator = $this
    ->container
    ->get('elcodi.entity_translator');
```

Once you have the service instance, you can set your entity translations in a
very simple way. In this example, let's create a new entity, let's save it into
database and let's translate it.

``` php
$product = new Product();
$product->setId(1);

$entityTranslator->save($product, [
    'es' => [
        'name' => 'el nombre',
        'description' => 'la descripción',
    ],
    'en' => [
        'name' => 'the name',
        'description' => 'the description',
        'anotherfield' => 'some value',
    ],
]);
```

## Translating an entity

So, your entity is translated? Well, let's load the translations of an entity.
You only need the entity and the language you want it to be translated to.

``` php
$product = $this
    ->get('doctrine.orm.entity_manager')
    ->find(1);

$entityTranslator->translate($product, 'en');
```

And that's it. Your product is now translated to english, and because both
fields, name and description, are not mapped anymore, you could flush the entity
manager and no information would be saved.

## Translations in forms

This is maybe the most important feature of this Bundle. How can we handle all
this stuff in our forms? Very simple. We don't have to change our form
definition but only adapt it to add an EventSubscriber.

Let's see an example.

``` php
use Elcodi\Component\EntityTranslator\EventListener\Traits\EntityTranslatableFormTrait;
use Symfony\Component\Form\AbstractType;

/**
 * Class ProductType
 */
class ProductType extends AbstractType
{
    use EntityTranslatableFormTrait;

    /**
     * Buildform function
     *
     * @param FormBuilderInterface $builder the formBuilder
     * @param array                $options the options for this form
     */
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder
            ->add('name', 'text', [
                'required' => true,
                'label'    => 'name',
            ])
            ->add('description', 'textarea', [
                'required' => false,
                'label'    => 'description',
            ]);

        $builder->addEventSubscriber($this->getEntityTranslatorFormEventListener());
    }

    /**
     * Return unique name for this form
     *
     * @return string
     */
    public function getName()
    {
        return 'form_type_product';
    }
}
```

As you can see, there are only two changes in this formType. First one is that
you must add this trait in order to add a protected variable and two methods.
The second change is that you must add the EventSubscriber the way is shown in
the example.

Because now the formType has some dependencies, you must define this formType as
a service in the dependency injection configuration.

``` yml
services:
    form_type.product:
        class: My\Namespace\Form\Type\ProductType
        calls:
            - [setEntityTranslatorFormEventListener, [@elcodi.entity_translator_form_event_listener]]
        tags:
            - { name: form.type, alias: form_type_product }
```

And that's it. Try to render your form, treating name and description like they
were normal fields and what you will see is that, instead of rendering a text
input field and a Textarea, three of each will be rendered.

## Custom form views

You can easily customize the way of rendering these fields by adding some format
into the rendering of `translatable_field`. You can find more information in
chapter - [What are Form Themes?](http://symfony.com/doc/current/cookbook/form/form_customization.html#what-are-form-themes)

## Automatic entity translation

You can enable the automatic translation of all loaded entities by enabling the
related configuration flag `auto_translate`.

``` yml
elcodi_entity_translation:
    auto_translate: true
```

By default this value is true. If true, every time you load a new entity from
Doctrine than is defined as translatable (a field is included in the
configuration), that entity will be translated using the current locale.
