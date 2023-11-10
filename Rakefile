# frozen_string_literal: true

require 'json'
require 'openssl'
require 'open-uri'
require 'yaml'
require 'time'
require 'nokogiri'
require 'quotable'
require 'liquid'
require 'dotenv/load'

GITHUB_USER = 'sobstel'
ALLOWED_REPO_ATTRS = %w[name full_name description html_url fork language stargazers_count]
EXCLUDED_REPOS = %w[AsyncHTTP Execution]

def save_data(name, object)
  file = "data/#{name}.yml"
  File.write(file, object.to_yaml)
  puts "saved to #{file}"
end

def load_data(name)
  file = "data/#{name}.yml"
  YAML.load_file(file, permitted_classes: [Time])
end

def github_fetch(url)
  puts "fetch #{url}"
  URI.open(url).read
end

def parse_repos(content)
  JSON.parse(content)
      .reject { |repo| repo['archived'] }
      .sort_by { |repo| repo['pushed_at'] }
      .collect { |repo| repo.select { |key, _| ALLOWED_REPO_ATTRS.include? key } }
      .reverse
end

desc 'Publish live'
task :publish, :message do |_, args|
  args.with_defaults message: Quotable.random.gsub(/[“”"]/, '')
  message = args[:message]

  # task(:import_github_repos).execute
  # task(:import_github_contributions).execute
  task(:generate_readmes).execute

  sh 'git add --all .'
  sh "git commit . --message=\"#{message}\""
  sh 'git pull --rebase'
  sh 'git push'
end

desc 'Generate READMEs'
task :generate_readmes do
  repos = load_data('repos')
  contribs = load_data('contribs')
  gists = load_data('gists')

  my_repos, forks = repos.partition { |repo| !repo['fork'] }

  popular_repos, other_repos = my_repos
    .select { |repo| repo['stargazers_count'] > 0 }
    .reject { |repo| EXCLUDED_REPOS.include?(repo['name']) }
    .partition { |repo| repo['stargazers_count'] >= 9 }

  template = Liquid::Template.parse(File.read('README.md.liquid'))
  content = template.render(
    'repos' => popular_repos + other_repos,
    'forks' => forks,
    'contribs' => contribs,
    'gists' => gists
  )
  File.write('README.md', content)

  template = Liquid::Template.parse(File.read('GISTS.md.liquid'))
  content = template.render(
    'gists' => gists
  )
  File.write('GISTS.md', content)
end

desc 'Import GitHub repos'
task :import_github_repos do
  repos = (1..2).flat_map do |page|
    url = format('https://api.github.com/users/%s/repos?page=%s', GITHUB_USER, page)
    parse_repos(github_fetch(url))
  end

  save_data('repos', repos)
end

desc 'Import GitHub contributions'
task :import_github_contributions do
  github_contributions = load_data('contribs')

  # last 12 months
  12.downto(0) do |i|
    from = DateTime.now - 30 * i
    to = DateTime.now - 30 * (i - 1)

    url = format('https://github.com/%s?tab=contributions&from=%s&to=%s', 'sobstel', from.to_s, to.to_s)
    puts "fetch from #{url}"

    html_doc = Nokogiri::HTML(github_fetch(url))
    contributions = html_doc.css('.contribution-activity-listing a[data-hovercard-type="repository"]')

    contributions.each do |contribution|
      github_contributions.unshift(contribution.text.strip)
    end
  end

  github_contributions.uniq!
  github_contributions.reject! { |repo| repo.start_with?('sobstel', 'golazon') }
  github_contributions.reject! { |repo| repo.include?('awesome') }

  save_data('contribs', github_contributions)
end

desc 'Import GitHub gists'
task :import_gists do
  gists = load_data('gists')
  since = gists.first['date'].to_s

  url = format('https://api.github.com/users/%s/gists?since=%s', 'sobstel', since)
  puts "fetch from #{url}"

  new_gists = JSON.parse(github_fetch(url)).select do |gist|
    gist['public']
  end

  new_gists.reject do |gist|
    gist['created_at'] <= since
  end.reverse.each do |gist|
    gists.unshift(
      'title' => gist['description'],
      'url' => gist['html_url'],
      'date' => gist['created_at']
    )
  end

  save_data('gists', gists)
end
