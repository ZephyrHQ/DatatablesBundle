# Examples

## Server side example

### Step 1: Create your Datatables class

```php
<?php

namespace Sg\BlogBundle\Datatables;

use Sg\DatatablesBundle\Datatable\View\AbstractDatatableView;

/**
 * Class PostDatatable
 *
 * @package Sg\BlogBundle\Datatables
 */
class PostDatatable extends AbstractDatatableView
{
    /**
     * {@inheritdoc}
     */
    public function buildDatatableView()
    {
        //-------------------------------------------------
        // Datatable
        //-------------------------------------------------

        $this->getFeatures()
            ->setServerSide(true)
            ->setProcessing(true);

        $this->getAjax()->setUrl($this->getRouter()->generate("post_results"));

        $this->getMultiselect()
            ->setEnabled(true)
            ->setPosition("last")
            ->addAction("Hide post", "post_bulk_disable")
            ->addAction("Delete post", "post_bulk_delete");

        $this->setStyle(self::BOOTSTRAP_3_STYLE);

        $this->setIndividualFiltering(true);


        //-------------------------------------------------
        // Columns
        //-------------------------------------------------

        $this->getColumnBuilder()
            ->add("id", "column", array(
                    "title" => "Post-id",
                    "searchable" => true,
                    "orderable" => true,
                    "visible" => true,
                    "class" => "active",
                    "width" => "100px"
                ))
            ->add("createdBy.username", "column", array(
                    "title" => "Created by"
                ))
            ->add("updatedBy.username", "column", array(
                    "title" => "Updated by"
                ))
            ->add("title", "column", array(
                    "title" => $this->getTranslator()->trans("test.title", array(), "msg")
                ))
            ->add("visible", "boolean", array(
                    "title" => "Visible",
                    "true_label" => "yes",
                    "false_label" => "no",
                    "true_icon" => "glyphicon glyphicon-ok",
                    "false_icon" => "glyphicon glyphicon-remove"
                ))
            ->add("createdAt", /*choose timeago or datetime*/ "datetime", array(
                    "title" => "Created at"
                ))
            ->add("tags.name", "array", array(
                    "title" => "Tags",
                    "read_as" => "tags[, ].name"
                ))
            ->add(null, "action", array(
                    "route" => "post_edit",
                    "parameters" => array(
                        "id" => "id"
                    ),
                    "renderif" => array(
                        "visible" // if this attribute is not NULL/FALSE
                    ),
                    "label" => $this->getTranslator()->trans("test.edit", array(), "msg"),
                    "attributes" => array(
                        "rel" => "tooltip",
                        "title" => "Edit User",
                        "class" => "btn btn-danger btn-xs"
                    ),
                ))
            ->add(null, "action", array(
                    "route" => "post_show",
                    "parameters" => array(
                        "id" => "id"
                    ),
                    "icon" => "glyphicon glyphicon-eye-open",
                    "label" => $this->getTranslator()->trans("test.show", array(), "msg"),
                    "attributes" => array(
                        "rel" => "tooltip",
                        "title" => "Show User",
                        "class" => "btn btn-primary btn-xs"
                    )
                ));
    }

    /**
     * {@inheritdoc}
     */
    public function getEntity()
    {
        return "SgBlogBundle:Post";
    }

    /**
     * {@inheritdoc}
     */
    public function getName()
    {
        return "post_datatable";
    }
}
```

### Step 2: Create your index.html.twig

```html
{% extends 'SgBlogBundle::layout.html.twig' %}

{% block content_content %}
    {{ datatable_render(datatable) }}
{% endblock %}
```

### Step 3: Registering your Datatables class as a Service

```yaml
services:

    sg_datatables.post:
        class: Sg\BlogBundle\Datatables\PostDatatable
        tags:
            - { name: sg.datatable.view }
```

### Step 4: Add controller actions

```php
/**
 * Post datatable.
 *
 * @Route("/", name="post")
 * @Method("GET")
 * @Template()
 *
 * @return array
 */
public function indexAction()
{
    $postDatatable = $this->get("sg_datatables.post");
    $postDatatable->buildDatatableView();

    return array(
        "datatable" => $postDatatable,
    );
}

/**
 * Get all Post entities.
 *
 * @Route("/results", name="post_results")
 *
 * @return \Symfony\Component\HttpFoundation\Response
 */
public function indexResultsAction()
{
    /**
     * @var \Sg\DatatablesBundle\Datatable\Data\DatatableData $datatable
     */
    $datatable = $this->get("sg_datatables.datatable")->getDatatable($this->get("sg_datatables.post"));

    // Callback example
    $function = function($qb)
    {
        $qb->andWhere("Post.visible = true");
    };

    // Add callback
    $datatable->addWhereBuilderCallback($function);

    return $datatable->getResponse();
}

/**
 * @Route("/bulk/delete", name="post_bulk_delete")
 * @Method("POST")
 *
 * @return Response
 */
public function bulkDeleteAction()
{
    $request = $this->getRequest();
    $isAjax = $request->isXmlHttpRequest();

    if ($isAjax) {
        $choices = $request->request->get("data");

        $em = $this->getDoctrine()->getManager();
        $repository = $em->getRepository("SgBlogBundle:Post");

        foreach ($choices as $choice) {
            $entity = $repository->find($choice["value"]);
            $em->remove($entity);
        }

        $em->flush();

        return new Response("This is ajax response.");
    }

    return new Response("This is not ajax.", 400);
}

/**
 * @Route("/bulk/disable", name="post_bulk_disable")
 * @Method("POST")
 *
 * @return Response
 */
public function bulkDisableAction()
{
    $request = $this->getRequest();
    $isAjax = $request->isXmlHttpRequest();

    if ($isAjax) {
        $choices = $request->request->get("data");

        $em = $this->getDoctrine()->getManager();
        $repository = $em->getRepository("SgBlogBundle:Post");

        foreach ($choices as $choice) {
            $entity = $repository->find($choice["value"]);
            $entity->setVisible(false);
            $em->persist($entity);
        }

        $em->flush();

        return new Response("This is ajax response.");
    }

    return new Response("This is not ajax.", 400);
}
```

## Non server side example

The differences to the above description:

### Your Datatables class

```php
<?php

namespace Sg\BlogBundle\Datatables;

use Sg\DatatablesBundle\Datatable\View\AbstractDatatableView;
use Sg\DatatablesBundle\Column\ActionColumn;

/**
 * Class PostDatatable
 *
 * @package Sg\BlogBundle\Datatables
 */
class PostDatatable extends AbstractDatatableView
{
    /**
     * {@inheritdoc}
     */
    public function buildDatatableView()
    {
        $this->getFeatures()->setServerSide(false);

        // ...
    }
}
```

### The controller action

```php
/**
 * Post datatable.
 *
 * @Route("/", name="post")
 * @Method("GET")
 * @Template()
 *
 * @return array
 */
public function indexAction()
{
    $repository = $this->getDoctrine()->getRepository('SgBlogBundle:Post');

    $query = $repository->createQueryBuilder('p')
        ->select('p, t, cb, ub')
        ->join('p.tags', 't')
        ->join('p.createdBy', 'cb')
        ->join('p.updatedBy', 'ub')
        ->getQuery();

    $results = $query->getArrayResult();

    $encoders = array(new JsonEncoder());
    $normalizers = array(new GetSetMethodNormalizer());
    $serializer = new Serializer($normalizers, $encoders);

    $postDatatable = $this->get('sg_datatables.post');
    $postDatatable->buildDatatableView();
    $postDatatable->setData($serializer->serialize($results, 'json'));

    return array(
        'datatable' => $postDatatable,
    );
}
```