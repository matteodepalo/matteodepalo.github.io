---
layout: post
title: "Refactor: Replace Method with Method Object"
date: 2013-02-01 11:40
comments: true
categories: [refactoring, ruby, rails]
---

In my previous [post](http://matteodepalo.github.com/blog/2013/01/31/how-to-create-custom-stylesheets-dynamically-with-rails-and-sass/) I described how to implement a feature that allows our customers to create custom stylesheets for their widget.
Altough it worked just fine, the `compile` class method of the `CustomTheme` class was blatantly big, so I decided to refactor it. 

The biggest issue I faced was that since this was a class method, in order to split it I should have created many little class methods and pass around the theme instance; a solution that didn't satisfy me. The reason `compile` needed to stay a class method is that I don't want to serialize the whole `CustomTheme` object and pass it to Sidekiq. Having considered this premises I could proceed in two ways:

* Delegate the class method `compile` to an instance method of a new custom theme, something along the lines of:

```ruby
def self.compile(theme_id)
  CustomTheme.find(theme_id).compile
end

private

def compile
  # perform the actual compilation
end
```

* Create a class with the name of the method and extract everything there (thanks [@bugant](https://twitter.com/bugant) for reminding me of this refactor)
 
I decided to go with the latter so I followed these steps:

1. Create the class ThemeCompiler
2. Give the new class an attribute for the object that hosted the original method (theme) and an attribute for each temporary variable in the method
3. Give the new class a method "compute"
4. Copy the body of the original method into compute
5. Split the compute method in smaller methods

## Final considerations

The first approach has the advantage of keeping everything in one class and use encapsulation properly, however it forces you to keep temp variables at the top of the compile method and increases the length of the class.

The second one puts every temp variables in the constructor but has the disadvantage of being envious of the `CustomTheme` class data to the point that it forces the promotion of one CustomTheme private method to public. Something like [friend classes](http://en.wikipedia.org/wiki/Friend_class) would have helped in this refactor.

The final result, indipendent of the methodology, is that the compile method is now much clearer.

## The code

```ruby
# custom_theme.rb

def self.compile(theme_id)
  ThemeCompiler.new(theme_id).compute
end
```

```ruby
# theme_compiler.rb
class ThemeCompiler
  attr_reader :theme, :body, :tmp_themes_path, :tmp_asset_name, :widget, :compressed_body, :asset, :env

  def initialize(theme_id)
    @theme = CustomTheme.find(theme_id)
    @body = ERB.new(File.read(File.join(Rails.root, 'app', 'assets', 'stylesheets', 'widget_custom.scss.erb'))).result(theme.get_binding)
    @tmp_themes_path = File.join(Rails.root, 'tmp', 'themes')
    @tmp_asset_name = theme.widget_id.to_s
    @widget = theme.widget
    @env = if Rails.application.assets.is_a?(Sprockets::Index)
      Rails.application.assets.instance_variable_get('@environment')
    else
      Rails.application.assets
    end
  end

  def compute
    create_temporary_file
    compile
    compress
    upload
  end

  private

  def compile
    @asset = env.find_asset(tmp_asset_name)
  rescue Sass::SyntaxError => error
    widget.user.notifications.create(:message => error.message.gsub(/ \(.+\)$/, ''), :type => 'error')
    theme.revert
  end

  def compress
    @compressed_body = ::Sass::Engine.new(asset.body, {
      :syntax => :scss,
      :cache => false,
      :read_cache => false,
      :style => :compressed
    }).render
  end

  def create_temporary_file
    FileUtils.mkdir_p(tmp_themes_path) unless File.directory?(tmp_themes_path)
    File.open(File.join(tmp_themes_path, "#{tmp_asset_name}.scss"), 'w') { |f| f.write(body) }
  end

  def upload
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
  end
end
```