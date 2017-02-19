#!/usr/bin/rake
require 'pathname'
require 'yaml'
require 'json'
require 'net/http'
require 'uri'



## [ Constants ] ##############################################################

BIN_NAME = 'swiftgen'
DEPENDENCIES = [:PathKit, :Stencil, :Commander, :StencilSwiftKit, :SwiftGenKit]
CONFIGURATION = 'Release'
BUILD_DIR = 'build/' + CONFIGURATION
TEMPLATES_SRC_DIR = 'Resources/templates'



## [ Utils ] ##################################################################

def version_select
  # Find all Xcode 8 versions on this computer
  xcodes = `mdfind "kMDItemCFBundleIdentifier = 'com.apple.dt.Xcode' && kMDItemVersion = '8.*'"`.chomp.split("\n")
  if xcodes.empty?
    raise "\n[!!!] You need to have Xcode 8.x to compile SwiftGen.\n\n"
  end
  # Order by version and get the latest one
  vers = lambda { |path| `mdls -name kMDItemVersion -raw "#{path}"` }
  latest_xcode_version = xcodes.sort { |p1, p2| vers.call(p1) <=> vers.call(p2) }.last
  %Q(DEVELOPER_DIR="#{latest_xcode_version}/Contents/Developer" TOOLCHAINS=com.apple.dt.toolchain.XcodeDefault.xctoolchain)
end

def xcpretty(cmd)
  if `which xcpretty` && $?.success?
    sh "set -o pipefail && #{cmd} | xcpretty -c"
  else
    sh cmd
  end
end

def xcrun(cmd)
  xcpretty "#{version_select} xcrun #{cmd}"
end

def print_info(str)
  (red,clr) = (`tput colors`.chomp.to_i >= 8) ? %W(\e[33m \e[m) : ["", ""]
  puts red, "== #{str.chomp} ==", clr
end

def defaults(args)
  bindir = args.bindir.nil? || args.bindir.empty? ? Pathname.new('./build/swiftgen/bin') : Pathname.new(args.bindir)
  fmkdir = args.fmkdir.nil? || args.fmkdir.empty? ? bindir + '../lib' : Pathname.new(args.fmkdir)
  tpldir = args.tpldir.nil? || args.tpldir.empty? ? bindir + '../templates' : Pathname.new(args.tpldir)
  [bindir, fmkdir, tpldir].map(&:expand_path)
end


## [ Build Tasks ] ############################################################

desc "Build the CLI binary and its frameworks in #{BUILD_DIR}"
task :build, [:bindir, :tpldir] => DEPENDENCIES.map { |dep| "dependencies:#{dep}" } do |_, args|
  (bindir, _, tpldir) = defaults(args)
  tpl_rel_path = tpldir.relative_path_from(bindir)
  main = File.read('SwiftGen/main.swift')
  File.write('SwiftGen/main.swift', main.gsub(/^let templatesRelativePath = .*$/, %Q(let templatesRelativePath = "#{tpl_rel_path}")))

  print_info "Building Binary"
  frameworks = DEPENDENCIES.map { |fmk| "-framework #{fmk}" }.join(" ")
  search_paths = DEPENDENCIES.map { |fmk| "-F #{BUILD_DIR}/#{fmk}" }.join(" ")
  xcrun %Q(-sdk macosx swiftc -O -o #{BUILD_DIR}/#{BIN_NAME} #{search_paths}/ #{frameworks} SwiftGen/*.swift)
end

namespace :dependencies do
  DEPENDENCIES.each do |fmk|
    # desc "Build #{fmk}.framework"
    task fmk do
      print_info "Building #{fmk}.framework"
      xcrun %Q(xcodebuild -project Pods/Pods.xcodeproj -target #{fmk} -configuration #{CONFIGURATION})
    end
end
end

desc "Build the CLI and link it so it can be run from #{BUILD_DIR}. Useful for testing without installing."
task :link => :build do
  sh %Q(install_name_tool -add_rpath "@executable_path" #{BUILD_DIR}/#{BIN_NAME})
end



## [ Install Tasks ] ##########################################################

desc "Install the binary in $bindir, frameworks — without the Swift dylibs — in $fmkdir, and templates in $tpldir\n" \
     "(defaults $bindir=./build/swiftgen/bin/, $fmkdir=$bindir/../lib, $tpldir=$bindir/../templates"
task 'install:light', [:bindir, :fmkdir, :tpldir] => :build do |_, args|
  (bindir, fmkdir, tpldir) = defaults(args)

  print_info "Installing binary in #{bindir}"
  sh %Q(mkdir -p "#{bindir}")
  sh %Q(cp -f "#{BUILD_DIR}/#{BIN_NAME}" "#{bindir}")

  print_info "Installing frameworks in #{fmkdir}"
  sh %Q(mkdir -p "#{fmkdir}")
  DEPENDENCIES.each do |fmk|
    sh %Q(cp -fr "#{BUILD_DIR}/#{fmk}/#{fmk}.framework" "#{fmkdir}")
  end
  sh %Q(install_name_tool -add_rpath "@executable_path/#{fmkdir.relative_path_from(bindir)}" "#{bindir}/#{BIN_NAME}")

  print_info "Installing templates in #{tpldir}"
  sh %Q(mkdir -p "#{tpldir}")
  sh %Q(cp -r "#{TEMPLATES_SRC_DIR}/" "#{tpldir}")
end

desc "Install the binary in $bindir, frameworks — including Swift dylibs — in $fmkdir, and templates in $tpldir\n" \
     "(defaults $bindir=./swiftgen/bin/, $fmkdir=$bindir/../lib, $tpldir=$bindir/../templates"
task :install, [:bindir, :fmkdir, :tpldir] => 'install:light' do |_, args|
  (bindir, fmkdir, tpldir) = defaults(args)

  print_info "Linking to standalone Swift dylibs"
  xcrun %Q(swift-stdlib-tool --copy --scan-executable "#{bindir}/#{BIN_NAME}" --platform macosx --destination "#{fmkdir}")
  toolchain_dir = `#{version_select} xcrun -find swift-stdlib-tool`.chomp
  xcode_rpath = File.dirname(File.dirname(toolchain_dir)) + '/lib/swift/macosx'
  xcrun %Q(install_name_tool -delete_rpath "#{xcode_rpath}" "#{bindir}/#{BIN_NAME}")
end



## [ Tests & Clean ] ##########################################################

desc "Run the Unit Tests"
task :tests do
  print_info "Running Unit Tests"
  xcrun %Q(xcodebuild -workspace SwiftGen.xcworkspace -scheme swiftgen -sdk macosx test)
end

desc "Delete the build/ directory"
task :clean do
  sh %Q(rm -fr build)
end

task :default => [:build]



## [ Playground Resources ] ###################################################

namespace :playground do
  task :clean do
    sh 'rm -rf SwiftGen.playground/Resources'
    sh 'mkdir SwiftGen.playground/Resources'
  end
  task :images do
    xcrun %Q(actool --compile SwiftGen.playground/Resources --platform iphoneos --minimum-deployment-target 7.0 --output-format=human-readable-text UnitTests/fixtures/Images.xcassets)
  end
  task :storyboard do
    xcrun %Q(ibtool --compile SwiftGen.playground/Resources/Wizard.storyboardc --flatten=NO UnitTests/fixtures/Storyboards-iOS/Wizard.storyboard)
  end
  task :strings do
    xcrun %Q(plutil -convert binary1 -o SwiftGen.playground/Resources/Localizable.strings UnitTests/fixtures/Localizable.strings)
  end

  desc "Regenerate all the Playground resources based on the test fixtures.\nThis compiles the needed fixtures and place them in SwiftGen.playground/Resources"
  task :resources => %w(clean images storyboard strings)
end



## [ Release a new version ] ##################################################

namespace :release do
  desc 'Create a new release on GitHub, CocoaPods and Homebrew'
  task :new => [:check_versions, :tests, :github, :cocoapods, :homebrew]

  def podspec_version(file = 'SwiftGen')
    JSON.parse(`bundle exec pod ipc spec #{file}.podspec`)["version"]
  end

  def log_result(result, label, error_msg)
    if result
      puts "#{label.ljust(25)} \u{2705}"
    else
      puts "#{label.ljust(25)} \u{274C}  - #{error_msg}"
    end
    result
  end

  desc 'Check if all versions from the podspecs and CHANGELOG match'
  task :check_versions do
    results = []

    # Check if bundler is installed first, as we'll need it for the cocoapods task (and we prefer to fail early)
    `which bundler`
    results << log_result( $?.success?, 'Bundler installed', 'Please install bundler using `gem install bundler` and run `bundle install` first.')

    # Extract version from GenumKit.podspec
    version = podspec_version
    puts "#{'SwiftGen.podspec'.ljust(25)} \u{1F449}  #{version}"

    # Check if entry present in CHANGELOG
    changelog_entry = system(%Q{grep -q '^## #{Regexp.quote(version)}$' CHANGELOG.md})
    results << log_result(changelog_entry, "CHANGELOG, Entry added", "Please add an entry for #{version} in CHANGELOG.md")

    changelog_master = system(%q{grep -qi '^## Master' CHANGELOG.md})
    results << log_result(!changelog_master, "CHANGELOG, No master", 'Please remove entry for master in CHANGELOG')

    # Check if example project updated
    sample_project_pods = YAML.load_file('Podfile.lock')['PODS']
    sample_project_updated = sample_project_pods.reduce(false) { |a, e| a || (e.is_a?(Hash) && e.keys.include?("GenumKit (#{version})")) }
    results << log_result(sample_project_updated, "Sample project updated", 'Please run `bundle exec pod update` on the sample project')

    exit 1 unless results.all?

    print "Release version #{version} [Y/n]? "
    exit 2 unless (STDIN.gets.chomp == 'Y')
  end

  desc 'Create a zip containing all the prebuilt binaries'
  task :zip => [:clean, :install] do
    `cp LICENSE README.md CHANGELOG.md build/swiftgen`
    `cd build/swiftgen; zip -r ../swiftgen-#{podspec_version}.zip .`
  end

  def post(url, content_type)
    uri = URI.parse(url)
    req = Net::HTTP::Post.new(uri, initheader = {'Content-Type' => content_type})
    yield req if block_given?
    req.basic_auth 'AliSoftware', File.read('.apitoken').chomp

    response = Net::HTTP.start(uri.host, uri.port, :use_ssl => (uri.scheme == 'https')) do |http|
      http.request(req)
    end
    unless response.code == '201'
      puts "Error: #{response.code} - #{response.message}"
      puts response.body
      exit 3
    end
    JSON.parse(response.body)
  end

  desc 'Upload the zipped binaries to a new GitHub release'
  task :github => :zip do
    v = podspec_version

    changelog = `sed -n /'^## #{v}$'/,/'^## '/p CHANGELOG.md`.gsub(/^## .*$/,'').strip
    print_info "Releasing version #{v} on GitHub"
    puts changelog

    json = post('https://api.github.com/repos/SwiftGen/SwiftGen/releases', 'application/json') do |req|
      req.body = { :tag_name => v, :name => v, :body => changelog, :draft => false, :prerelease => false }.to_json
    end

    upload_url = json['upload_url'].gsub(/\{.*\}/,"?name=swiftgen-#{v}.zip")
    zipfile = "build/swiftgen-#{v}.zip"
    zipsize = File.size(zipfile)

    print_info "Uploading ZIP (#{zipsize} bytes)"
    post(upload_url, 'application/zip') do |req|
      req.body_stream = File.open(zipfile, 'rb')
      req.add_field('Content-Length', zipsize)
      req.add_field('Content-Transfer-Encoding', 'binary')
    end
  end

  desc 'pod trunk push SwiftGen to CocoaPods'
  task :cocoapods do
    print_info "Pushing pod to CocoaPods Trunk"
    sh 'bundle exec pod trunk push SwiftGen.podspec'
  end

  desc 'Release a new version on Homebrew and prepare a PR'
  task :homebrew do
    print_info "Updating Homebrew Formula"
    tag = podspec_version
    formulas_dir = `brew --repository homebrew/core`.chomp
    Dir.chdir(formulas_dir) do
      sh 'git checkout master'
      sh 'git pull'
      sh "git checkout -b swiftgen-#{tag} origin/master"

      targz_url = "https://github.com/SwiftGen/SwiftGen/archive/#{tag}.tar.gz"
      sha256_res = `curl -L #{targz_url} | shasum -a 256`
      sha256 = /^[A-Fa-f0-9]+/.match(sha256_res)
      raise 'Unable to extract SHA256' if sha256.nil?
      formula_file = "#{formulas_dir}/Formula/swiftgen.rb"
      formula = File.read(formula_file)
      new_formula = formula.gsub(/url "https:.*"$/, %Q(url "#{targz_url}")).gsub(/sha256 ".*"$/,%Q(sha256 "#{sha256.to_s}"))
      File.write(formula_file, new_formula)

      print_info "Checking Homebrew formula..."
      sh 'brew audit --strict --online swiftgen'
      sh 'brew upgrade swiftgen'
      sh 'brew test swiftgen'

      print_info "Pushing to Homebrew"
      sh "git add #{formula_file}"
      sh "git commit -m 'swiftgen #{tag}'"
      sh "git push -u AliSoftware swiftgen-#{tag}"
      sh "open 'https://github.com/Homebrew/homebrew-core/compare/master...AliSoftware:swiftgen-#{tag}?expand=1'"
    end
  end

end
