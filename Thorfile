class BuildWebsite < Thor
  include Thor::Actions
  namespace :default
  source_root "_templates"

  attr_reader :title

  desc "build", "Generate the site"
  def build
    system "cd _sass && bundle exec compass compile"
    system "bundle exec jekyll"
  end

  desc "watch", "Observe changes and update accordingly"
  def watch
    shell.mute do
      run "trap 'kill -INT $JEKYLL_PID $COMPASS_PID' exit
           bundle exec jekyll --auto & JEKYLL_PID=$!
           cd _sass && bundle exec compass watch & COMPASS_PID=$!
           wait"
    end
  end

  desc "serve", "Run a Rack server of the content, updated live"
  def serve
    shell.mute do
      run "trap 'kill -INT $JEKYLL_PID $COMPASS_PID' exit
           bundle exec jekyll --auto --server & JEKYLL_PID=$!
           cd _sass && bundle exec compass watch & COMPASS_PID=$!
           wait"
    end
  end

  desc "upload", "Upload to Google Storage"
  def upload
    shell.mute do
      run "gsutil -m cp -a public-read -z html,css,vcf,js,pdf,xml -R _site/* gs://www.jeremyroman.com/"
    end
  end

  desc "clean", "Clean up temporary stuff"
  def clean
    run "rm -rf .sass-cache/**"
    remove_dir "_site"
    remove_file "stylesheets/jeremyroman.css"
    empty_directory "_site"
  end

  desc "new TITLE", "Create a new post"
  method_options :edit => true
  def new(title)
    @title = title
    today = Time.now.strftime("%Y-%m-%d")
    slug = title.downcase.gsub(/["'.!?]/, "").gsub(/[^a-z0-9]+/, "-").gsub(/^-+|-+$/, "")
    filename = "_posts/#{today}-#{slug}.md"

    template "post.md", filename

    if options[:edit] and File.exists?("/usr/local/bin/mvim")
      run "mvim '#{filename}'"
    elsif options[:edit]
      run "vim '#{filename}'"
    end
  end
end
