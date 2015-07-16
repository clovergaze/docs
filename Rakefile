# Directories and files generated by this Rakefile
#   - _includes: Jekyll includes
#   - assets/images/docs: Images for the docs repo
$generated_files = [
    '_data',
    '_includes',
    '_layouts',
    '_plugins',
    'assets/',
    'assets/images/docs',
]

# ---- Rake tasks
task :default => :serve

desc 'Set up the build environment'
task :init do
    # Install packages
    sh 'bundle install'
end

# Copy assets needed for the Jekyll build
desc 'Copy assets and includes for the Jekyll build'
task :copy_assets do
    # Create each destination directory, if it doesn't already exist
    ['_data/docs','_includes','assets/images/docs'].each{ |dir_name|
        FileUtils.mkdir_p(dir_name) unless Dir.exists?(dir_name)
    }

    assets_to_copy = [
        {:source => '_jekyll/_standalone/assets/.', :target => 'assets/'},
        {:source => '_jekyll/_images/.', :target => 'assets/images/docs/'},
        {:source => '_jekyll/_data/.', :target => '_data/docs/'},
        {:source => '_jekyll/_includes/.', :target => '_includes/docs/'},
        {:source => '_jekyll/_standalone/_layouts/.', :target => '_layouts/'},
        {:source => '_jekyll/_standalone/_plugins/.', :target => '_plugins/'},
    ]
    assets_to_copy.each{ |asset|
        FileUtils.cp_r(asset[:source], asset[:target], :verbose => true)
    }
end

task :rewrite do
    require 'yaml'
    root = '_site'

    # Get the RethinkDB version from the config
    config = YAML.load_file('_config.yml')
    version = config['version']
    base = "/#{version}"

    puts "RethinkDB documentation version: #{version}"

    # Rewrite all resources to use the new baseurl
    rewrite_html(root, baseurl=base)
    rewrite_css("#{root}/assets/css/styles.css", baseurl=base)
    rewrite_js("#{root}/assets/js/site.js", baseurl=base)

    # Index the files we need to relocate -- to a version-specific folder
    files = Dir.glob("#{root}/*")

    # Create the new folder, move all the files to it
    new_path = "#{root}/#{version}/"

    FileUtils.mkdir(new_path)
    FileUtils.mv(files, new_path)
end

desc 'Clean up the generated site'
task :clean do
    rm_rf '_site'
    rm '.jekyll-metadata', :force => true
    $generated_files.each{ |d|
        rm_rf d
    }
end

desc 'Build site with Jekyll'
task :build => ['clean', 'copy_assets'] do
    jekyll('build')
end

desc 'Start server and regenerate files on change'
task :serve => ['copy_assets'] do
    check_for_required_files(:warning => true)
    jekyll('serve') 
end

# ---- Rake functions

# Run Jekyll
def jekyll(opts='')
    if ENV['dev']=='on'
        dev = ' --plugins=_plugins,_plugins-dev'
    else
        dev = ''
    end
    sh "bundle exec jekyll #{opts}#{dev} --trace"
end

# Check if all generated files are present: by default abort if files aren't present, otherwise show a warning
def check_for_required_files(opts={})
    missing_files = 0
    $generated_files.each do |f|
        if !File.exists?(f)
            puts "Required file missing: #{f}"
            missing_files +=1
        end
    end
    if missing_files > 0
        error = "#{missing_files} required files not found. Run `rake build` before deploying."
        if opts[:warning] then puts error else fail error end
    end
end

# Prepends all linked resources with the specified baseurl
def rewrite_html(root='', baseurl='')
    require 'nokogiri'
    require 'tempfile'

    num_docs = 0
    num_links = 0

    # Prepends a link with the baseurl, if it starts with the right string
    # Proc takes two arguments:
    #   - link: the link to rewrite
    #   - starts_with: only rewrite the link if it starts with the given string
    #     (defaults to '/')
    rewrite = Proc.new do |link, starts_with='/'|
        if link and link.start_with?(starts_with)
            num_links += 1
            "#{baseurl}#{link}"
        else
            "#{link}"
        end
    end

    puts "Adding baseurl to all links... (using baseurl: #{baseurl})"
    Dir.glob(root + '/**/*.html') do |path|
        num_docs += 1
        doc = Nokogiri::HTML(open(path))

        # Rewrite links
        doc.css('a').each do |a|
            if a.key?('href')
                # Exclude anchor links
                a['href'] = rewrite.call(a['href'])
            end
        end

        # Rewrite image paths
        doc.css('img').each do |img|
            if img.key?('src')
                img['src'] = rewrite.call(img['src'])
            end
        end

        # Rewrite JavaScript file paths
        doc.css('script').each do |script|
            if script.key?('src')
                script['src'] = rewrite.call(script['src'], starts_with='/assets')
            end
        end

        # Rewrite CSS stylesheet paths
        doc.css('link').each do |link|
            if link.key?('href')
                link['href'] = rewrite.call(link['href'], starts_with='/assets')
            end
        end

        doc.write_to(open(path, 'w'))
    end
    puts "Processed #{num_docs} documents and rewrote #{num_links} links."
end

# Rewrite resources in a CSS file to start with a given baseurl
#   - css_file: the file to rewrite
#   - baseurl: the baseurl to prepend all links with
def rewrite_css(css_file, baseurl='') 
    require 'tempfile'
    num_changes = 0

    # Create a temporary file
    temp = Tempfile.new('styles.css')

    # Rewrite each line of the CSS file, output to the temp file
    File.open(css_file, 'r') do |f|
        f.each_line do |line|
            # Check if each line contains the substring:
            #   - url(<match>)
            #   - url("<match>")
            #   - url('<match>')
            # Two capture groups are used:
            #   - url("
            #   - <link url>")
            # ...and the baseurl is inserted between the two
            line.gsub!(/(url\("?'?)([^")]*"?\))/) do
                num_changes += 1 
                "#{$1}#{baseurl}#{$2}"
            end
            temp.puts line
        end
    end
    temp.close()

    # Replace the existing CSS file with the rewritten file
    FileUtils.mv(temp.path, css_file)

    puts "Rewrote CSS file, #{num_changes} links updated: #{css_file}"
end

# Rewrites one specific file: the site.js file used on the RethinkDB website
# for interaction and functionality
def rewrite_js(js_file, baseurl='')
    require 'tempfile'
    num_changes = 0

    # Create a temporary file
    temp = Tempfile.new('script.js')
    
    # Rewrite each line of the JS file, output to the temp file
    File.open(js_file, 'r') do |f|
        is_rewriting = false
        f.each_line do |line|
            # Don't rewrite until we get to one specific section
            if line['routes = {']
                is_rewriting = true
            end

            if is_rewriting
                # Prepend each route with the baseurl.
                # Two capture groups are used:
                #   - opening quote
                #   - the rest of the line
                # ...in between which a baseurl is inserted
                line.gsub!(/(')(.*)/) do
                    num_changes += 1
                    "#{$1}#{baseurl}#{$2}"
                end
                # Once we reach the end of the section, stop rewriting
                if line['}']
                    is_rewriting = false
                end
            end
            temp.puts line
        end
    end
    temp.close()

    # Replace the existing JS file with the rewritten file
    FileUtils.mv(temp.path, js_file)

    puts "Rewrote JS file, #{num_changes} routes updated: #{js_file}"

end
