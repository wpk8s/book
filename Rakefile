namespace :book do
  def exec_or_raise(command)
    puts `#{command}`
    if (! $?.success?)
      raise "'#{command}' failed"
    end
  end

  # Variables referenced for build
  version_string = ENV['TRAVIS_TAG'] || `git describe --tags`.chomp
  if version_string.empty?
    version_string = '0'
  end
  date_string = Time.now.strftime('%Y-%m-%d')
  params = "--attribute revnumber='#{version_string}' --attribute revdate='#{date_string}'"
  header_hash = `git rev-parse --short HEAD`.strip

  # Check contributors list
  # This checks commit hash stored in the header of list against current HEAD
  def check_contrib
    if File.exist?('book/contributors.txt')
      current_head_hash = `git rev-parse --short HEAD`.strip
      header = `head -n 1 book/contributors.txt`.strip
      # Match regex, then coerce resulting array to string by join
      header_hash = header.scan(/[a-f0-9]{7,}/).join

      if header_hash == current_head_hash
        puts "Hash on header of contributors list (#{header_hash}) matches the current HEAD (#{current_head_hash})"
      else
        puts "Hash on header of contributors list (#{header_hash}) does not match the current HEAD (#{current_head_hash}), refreshing"
        `rm book/contributors.txt`
        # Reenable and invoke task again
        Rake::Task['book/contributors.txt'].reenable
        Rake::Task['book/contributors.txt'].invoke
      end
    end
  end

  desc 'build basic book formats'
  task :build => [:build_html, :build_epub, :build_pdf] do
    begin
        # Run check
        Rake::Task['book:check'].invoke

        # Rescue to ignore checking errors
        rescue => e
        puts e.message
        puts 'Error when checking books (ignored)'
    end
  end

  desc 'build basic book formats (for ci)'
  task :ci => [:build_html, :build_epub, :build_pdf] do
      # Run check, but don't ignore any errors
      Rake::Task['book:check'].invoke
  end

  desc 'generate contributors list'
  file 'book/contributors.txt' do
      puts 'Generating contributors list'
      `echo "Contributors as of #{header_hash}:\n" > book/contributors.txt`
      `git shortlog -s | grep -v -E "(Madalin|Mădălin|dependabot)" | cut -f 2- | column -c 120 >> book/contributors.txt`
  end

  desc 'build HTML format'
  task :build_html => 'book/contributors.txt' do
      check_contrib()

      puts 'Converting to HTML...'
      `bundle exec asciidoctor #{params} -a data-uri wpk8s.adoc`
      puts ' -- HTML output at wpk8s.html'

  end

  desc 'build Epub format'
  task :build_epub => 'book/contributors.txt' do
      check_contrib()

      puts 'Converting to EPub...'
      `bundle exec asciidoctor-epub3 #{params} wpk8s.adoc`
      puts ' -- Epub output at wpk8s.epub'

  end

  desc 'build Mobi format'
  task :build_mobi => 'book/contributors.txt' do
      # Commented out the .mobi file creation because the kindlegen dependency is not available.
      # For more information on this see: #1496.
      # This is a (hopefully) temporary fix until upstream asciidoctor-epub3 is fixed and we can offer .mobi files again.

      # puts "Converting to Mobi (kf8)..."
      # `bundle exec asciidoctor-epub3 #{params} -a ebook-format=kf8 wpk8s.adoc`
      # puts " -- Mobi output at wpk8s.mobi"

      # FIXME: If asciidoctor-epub3 supports Mobi again, uncomment these lines below
      puts "Converting to Mobi isn't supported yet."
      puts "For more information see issue #1496 at https://github.com/wpk8s/wpk8s2/issues/1496."
      exit(127)
  end

  desc 'build PDF format'
  task :build_pdf => 'book/contributors.txt' do
      check_contrib()

      puts 'Converting to PDF... (this one takes a while)'
      `bundle exec asciidoctor-pdf -a pdf-theme=theme.yml -a pdf-fontsdir=fonts wpk8s.adoc`
      puts ' -- PDF output at wpk8s.pdf'
  end

  desc 'Check generated books'
  task :check => [:build_html, :build_epub] do
      puts 'Checking generated books'

      exec_or_raise('htmlproofer --check-html --disable-external wpk8s.html')
      exec_or_raise('epubcheck wpk8s.epub')
  end

  desc 'Clean all generated files'
  task :clean do
    begin
        puts 'Removing generated files'

        FileList['book/contributors.txt', 'wpk8s.html', 'wpk8s.epub', 'wpk8s.pdf'].each do |file|
            rm file

            # Rescue if file not found
            rescue Errno::ENOENT => e
              begin
                  puts e.message
                  puts 'Error removing files (ignored)'
              end
        end
    end
  end

end

task :default => "book:build"
