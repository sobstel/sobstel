# frozen_string_literal: true

require 'json'
require 'openssl'
require 'open-uri'
require 'yaml'
require 'time'
require 'nokogiri'
require 'gemoji-parser'
require 'quotable'
require 'liquid'
require 'dotenv/load'

def save_data(name, object)
  file = "data/#{name}.yml"
  File.write(file, object.to_yaml)
  puts "saved to #{file}"
end

def load_data(name)
  file = "data/#{name}.yml"
  YAML.load_file(file)
end

def github_fetch(url)
  options = {
    http_basic_authentication: ['sobstel', ENV['GITHUB_PAT']],
    ssl_verify_mode: OpenSSL::SSL::VERIFY_NONE
  }
  URI.open(url, options).read
end

def fetch_repos(url)
  allowed_repo_attrs = %w[name full_name description html_url fork languages stargazers_count]

  JSON.parse(github_fetch(url))
      .reject { |repo| repo['archived'] }
      .sort_by { |repo| repo['pushed_at'] }
      .collect { |repo| repo.select { |key, _| allowed_repo_attrs.include? key } }
      .reverse
  #       .tap { |repo| repo['description'] = EmojiParser.detokenize(repo['description']) if repo['description'] }
end

desc 'Publish live'
task :publish, :message do |_, args|
  args.with_defaults message: Quotable.random.gsub(/[“”]/, '')
  message = args[:message]

  # task(:import_github_repos).execute
  # task(:import_gists).execute
  # task(:import_github_contributions).execute

  sh 'git add --all .'
  sh "git commit . --message=\"#{message}\""
  sh 'git pull --rebase'
  sh 'git push'
end

desc 'Generate README'
task :generate_readme do
  template = Liquid::Template.parse(File.read('README.md.liquid'))
  content = template.render(
    'repos' => load_data('repos'),
    'contribs' => load_data('contribs'),
  )
  File.write('README.md', content)
end

desc 'Import GitHub repos'
task :import_github_repos do
  url = format('https://api.github.com/users/%s/repos', 'sobstel')
  puts "fetch from #{url}"

  repos = fetch_repos(url).sort_by { |repo| repo['stargazers_count'] }.reverse
  save_data('repos', repos)

  # repos, forks = all_repos.partition { |repo| !repo['fork'] }
  # popular_repos, other_repos = repos.partition do |repo|
  #   next repo['stargazers_count'] >= 9
  # end

  # save_data('popular_repos', popular_repos)
  # save_data('other_repos', other_repos)
  # save_data('forks', forks)
end


desc 'Import GitHub contributions'
task :import_github_contributions do
  github_contributions = load_data('contribs')

  # last 3 months
  3.downto(0) do |i|
    from = DateTime.now - 30 * i
    to = DateTime.now - 30 * (i - 1)

    url = format('https://github.com/%s?tab=contributions&from=%s&to=%s', 'sobstel', from.to_s, to.to_s)
    puts "fetch from #{url}"

    html_doc = Nokogiri::HTML(github_fetch(url))
    contributions = html_doc.css('.contribution-activity-listing a[data-hovercard-type="repository"]')

    contributions.each do |contribution|
      github_contributions << contribution.text.strip
    end
  end

  github_contributions.uniq!
  github_contributions.reject! { |repo| repo.start_with?('sobstel', 'golazon', 'hydropuzzle') }
  github_contributions.sort_by!(&:downcase)

  save_data('contribs', github_contributions)
end
