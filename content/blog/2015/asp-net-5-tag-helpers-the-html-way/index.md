---
title: "ASP.NET 5 - Tag Helpers, the HTML way"
date: 2015-05-14T00:00:00+00:00
publishdate: 2015-05-14T00:00:00+00:00
lastmod: 2015-05-14T00:00:00+00:00
tags: ["ASP.NET", "MVC"]
comments: true
draft: false
disableBreadcrumb: true
---

<h4>Introduction</h4>
<p>Tag Helper is not a new concept in ASP.NET but they way we use Tag Helpers in razor view is different. In the previous versions of ASP.NET, if we want to use HTML tag helper, we will be switching from HTML's native declarative style to imperative style of C#.</p><!-- more -->
<p>We loose the tooling support we get when editing HTML document for the most part. For example, the following code creates a label</p>

```cs
// Old style tag helpers
@Html.LabelFor(Model => Model.Name, "User Name", new { @class = "control-label" });
```

<p>In this example, we are specifying that this label is created for Name property of the Model using lambda expression, then label text as second string parameter and finally we are creating an anonymous type for all the html attributes we want to attach to the tag. When we create the label this way we won't get any support from Visual Studio that we get when authoring html document, especially, for the last parameter, because Visual Studio won't know whether we are creating HTML attributes or something else.</p>
<h4>Declarative Style</h4>
<p>In ASP.NET 5, we create the same label like this,</p>

```cs
// Declarative style
<label asp-for="Name" class="control-label">User Name</label>
```

<p>Note, we have a new attribute called <code>asp-for</code>, this replaces the lambda expression we used in the previous example and the rest is self explanatory. One of the main advantage of this style is that we get full intellisense and other supports from Visual Studio that we usually get while authoring html document.</p>
<h4>Built-in Tag Helpers</h4>
<p>ASP.NET 5 has many built-in tag helpers and they are in Microsoft.AspNet.Mvc.TagHelpers assembly. We can use the built-in tag helpers by importing them into the scope using a new directive @addTagHelper like this,</p>

```cs
// add tag helpers
@addTagHelper "*, Microsoft.AspNet.Mvc.TagHelpers"
```

<p>We specify type name(s) followed by assembly name separated by comma as a string. * specifies all the types in the assembly. </p>
<h4>Custom Tag Helpers</h4>
<p>ASP.NET 5 provides a very simple interface ITagHelper to create custom tag helpers. The interface has only two members,</p>

```cs
// ITagHelper
int Order { get; }

Task ProcessAsync(TagHelperContext context, TagHelperOutput output);
```

<p>The ProcessAsyc method takes two parameters, the first one provides options to get all the information about the tag in question and the helper itself and the output parameter represents the output that will be rendered. We can manipulate output to influence the content rendered.</p>
<p>The Order property indicates the order in which the ProcessAsync will be executed on the target tag where there are more then one tag helpers target the same tag. The Tag helper process will be applied from lower order to higher order.</p>
<p>We have a class TagHelper which implements the ITagHelper, so if we want to create new tag helper, we just need to inherit our class from TagHelper and override Order and ProcessAsync method. Note, the convention is to a tag helper with 'TagHelper' suffix.</p>
<p>Here is an example of custom tag helper which creates bootstrap panel</p>

```cs
    // Custom tag helper example
    [TargetElement("div", Attributes = "panel-title")]
    [TargetElement("div", Attributes = "panel-style")]
    public class PanelTagHelper : TagHelper
    {
        [HtmlAttributeName("panel-title")]
        public string Title { get; set; }

        [HtmlAttributeName("panel-type")]
        public HtmlPanelType PanelType { get; set; }

        public override void Process(TagHelperContext context, TagHelperOutput output)
        {
            //<div class="panel panel-default">
            //  <div class="panel-heading">
            //    <h3 class="panel-title">Panel title</h3>
            //  </div>
            //  <div class="panel-body">
            //    Panel content
            //  </div>
            //</div>

            // header content
            var panelHeadingContent = new TagBuilder("h3");
            panelHeadingContent.SetInnerText(Title);

            // header
            var panelHeading = new TagBuilder("div");
            panelHeading.AddCssClass("panel-heading");
            panelHeading.InnerHtml = panelHeadingContent.ToHtmlString(TagRenderMode.Normal).ToString();

            // body
            var panelBody = new TagBuilder("div");
            panelBody.AddCssClass("panel-body");
            panelBody.InnerHtml = context.GetChildContentAsync().Result.GetContent();

            // panel
            var panel = new TagBuilder("div");
            panel.AddCssClass("panel");
            AddPanelTypeStyle(panel, PanelType);

            // replace the custom tag with the panel just built
            output.MergeAttributes(panel);
            var content = panelHeading.ToHtmlString(TagRenderMode.Normal).ToString();
            content += panelBody.ToHtmlString(TagRenderMode.Normal).ToString();
            output.Content.SetContent(content);

            base.Process(context, output);
        }

        private void AddPanelTypeStyle(TagBuilder panel, HtmlPanelType type)
        {
            switch (type)
            {
                case HtmlPanelType.Default:
                    panel.AddCssClass("panel-default");
                    break;

                case HtmlPanelType.Primary:
                    panel.AddCssClass("panel-primary");
                    break;

                case HtmlPanelType.Info:
                    panel.AddCssClass("panel-info");
                    break;

                case HtmlPanelType.Success:
                    panel.AddCssClass("panel-success");
                    break;

                case HtmlPanelType.Warning:
                    panel.AddCssClass("panel-warning");
                    break;

                case HtmlPanelType.Danger:
                    panel.AddCssClass("panel-danger");
                    break;

                default:
                    throw new InvalidOperationException("The panel type specified is not specified or invalid");
            }
        }

    }
```

<p><br />The TargetElement attribute is used to specify the target element to which this tag helper applies. We can specify tag name and optional attributes. We can also specify more than one TargetElement attribute and they work in logical OR basis.</p>
<p>The HtmlAttributeName attribute is used to bind an attribute of a tag with property in the tag helper. If you don't want the property in tag helper to represent any html attributes then you decorate the property with HtmlAttributeNotBoundAttribute to indicate that.</p>

{{% notice note%}}
ASP.NET 5 is in beta at the time this post is published and the final version may be different than the beta version.
{{% /notice %}}