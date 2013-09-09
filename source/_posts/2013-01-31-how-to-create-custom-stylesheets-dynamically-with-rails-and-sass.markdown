---
layout: post
title: "How to create custom stylesheets dynamically with Rails and Sass"
date: 2013-01-31 12:24
comments: true
categories: [sass, sprockets, rails]
alias: [/blog/2013/01/31/how-to-create-dynamic-stylesheets-with-rails-and-sass]
---

At [Responsa](http://goresponsa.com) we have the need to create custom stylesheets for our widget administrators. In order to accomplish this we leverage the power of Sass and the Rails asset pipeline.

In this blog post I'll show you how we implemented this feature and how to deploy it to an Heroku + Amazon S3 production environment.

## Tools
Let's take a loot at our toolbelt:

- Sass and Sprockets to dynamically compile the asset
- Sidekiq to delay the compilation and upload to S3, which in our case takes between 10 and 15 seconds
- Fog gem to store on S3

## Models
We have 2 models: Widget and CustomTheme

```ruby
class CustomTheme
  include Mongoid::Document

  belongs_to :widget

  field :main_color, :type => String, :default => "#2ba6cb"
  field :text_font, :type => String, :default => "\"Helvetica Neue\", \"Helvetica\", Helvetica, Arial, sans-serif"
  field :digest, :type => String
end
```

```ruby
class Widget
  include Mongoid::Document

  has_one :custom_theme
end
```

The custom theme model has the fields used in a widget_custom.scss stylesheet built with the [Foundation](http://foundation.zurb.com) CSS framework:

```css
$mainColor: <%= main_color %>;
$bodyFontFamily: <%= text_font %>;

@import "widget/index";
```

## Compilation

CustomTheme has a method we call every time we need to compile a fresh asset which occurs when the fields change. It performs a few actions in order:

1. Write a temporary and not compiled scss file with the variables taken from the custom theme and give it a unique name.
2. Use the Sprockets environment to find this temporary file and compile it.
3. Compress the compiled css file.
4. Store it either on amazon S3 or the file system.
5. Delete the previous asset.

## Caveats

Developing this solution we encountered a few problems mainly due to our production setup and the way Sprockets works:

- If the compilation fails we need to restore the previous asset. To accomplish this we basically keep track of the previous asset and revert to it if anything goes wrong.
- In production we need to avoid using the cached Sprockets environment, else Sprockets will cache the entire file system at the beginning.
- It's important to run validations of the custom theme fields in order to avoid css injection.

## Code

```ruby
# application.rb
config.assets.paths << Rails.root.join('tmp', 'themes')
```

```ruby
COMPILED_FIELDS = [:main_color, :text_font]

after_save :compile, :if => :compiled_attributes_changed?

def self.compile(theme_id)
  theme = CustomTheme.find(theme_id)
  body = ERB.new(File.read(File.join(Rails.root, 'app', 'assets', 'stylesheets', 'widget_custom.scss.erb'))).result(theme.get_binding)
  tmp_themes_path = File.join(Rails.root, 'tmp', 'themes')
  tmp_asset_name = theme.widget_id.to_s

  FileUtils.mkdir_p(tmp_themes_path) unless File.directory?(tmp_themes_path)
  File.open(File.join(tmp_themes_path, "#{tmp_asset_name}.scss"), 'w') { |f| f.write(body) }

  widget = theme.widget

  begin
    env = if Rails.application.assets.is_a?(Sprockets::Index)
      Rails.application.assets.instance_variable_get('@environment')
    else
      Rails.application.assets
    end

    asset = env.find_asset(tmp_asset_name)

    compressed_body = ::Sass::Engine.new(asset.body, {
      :syntax => :scss,
      :cache => false,
      :read_cache => false,
      :style => :compressed
    }).render

    theme.delete_asset

    if Rails.env.production?
      FOG_STORAGE.directories.get(ENV['FOG_DIRECTORY']).files.create(
        :key    => theme.asset_path(asset.digest),
        :body   => StringIO.new(compressed_body),
        :public => true,
        :content_type => 'text/css'
      )
    else
      File.open(File.join(Rails.root, 'public', theme.asset_path(asset.digest)), 'w') { |f| f.write(compressed_body) }
    end

    theme.update_attribute(:digest, asset.digest)
  rescue Sass::SyntaxError => error
    theme.revert
  end

  widget.save
end

def revert
  # Revert to your previous theme and notify the user of the failure
end

def get_binding
  binding
end

def delete_asset
  return unless digest?

  if Rails.env.production?
    FOG_STORAGE.directories.get(ENV['FOG_DIRECTORY']).files.get(asset_path).try(:destroy)
  else
    File.delete(File.join(Rails.root, 'public', asset_path))
  end
end

def asset_path(digest)
  "assets/themes/#{asset_name(digest)}.css"
end

def asset_name(digest = self.digest)
   "#{widget_id}-#{digest}"
end

def asset_url
  "#{ActionController::Base.asset_host}/#{asset_path}"
end

private

def compile
  self.class.delay.compile(id)
end

def compiled_attributes_changed?
  changed_attributes.keys.map(&:to_sym).any? { |f| COMPILED_FIELDS.include?(f) }
end
```

Finally we use the asset url in our template:

```ruby
<%= stylesheet_link_tag @custom_theme.asset_url %>
```
