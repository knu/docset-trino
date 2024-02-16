# frozen_string_literal: true
require 'bundler/setup'
Bundler.require

require 'json'
require 'pathname'
require 'tempfile'
require 'uri'
require 'rubygems/version'

def jenkins?
  /jenkins-/.match?(ENV['BUILD_TAG'])
end

def capture(*args)
  `#{args.flatten.shelljoin}`
end

def paginate_command(cmd, diff: false)
  case cmd
  when Array
    cmd = cmd.shelljoin
  end

  if $stdout.tty? || (diff && jenkins?)
    pager = (ENV['DIFF_PAGER'] if diff) || ENV['PAGER'] || 'more'
    "#{cmd} | #{pager}"
  else
    cmd
  end
end

def git_ref_exists?(ref)
  system(*%W[git show-ref --quiet origin/#{DUC_BRANCH}])
end

def text_node_match(node, pattern)
  case node
  when Nokogiri::XML::Text
    pattern.match(node.text)
  end
end

def between_texts?(node, prev_pattern, next_pattern)
  !!(text_node_match(node.previous, prev_pattern) &&
     text_node_match(node.next, next_pattern))
end

def extract_version
  cd DOCS_ROOT do
    Dir.glob('**/index.html') { |path|
      doc = Nokogiri::HTML(File.read(path), path)
      if version = doc.title[/Trino ([\d.]+)/, 1]
        return version
      end
    }
  end
  nil
end

def existing_pull_url
  repo = "#{DUC_OWNER_UPSTREAM}/#{DUC_REPO}"
  query = "repo:#{repo} is:pr author:#{DUC_OWNER} head:#{DUC_BRANCH} is:open"
  graphql = <<~GRAPHQL
    query($q: String!) {
      search(
        query: $q
        type: ISSUE
        last: 1
      ) {
        edges {
          node {
            ... on PullRequest {
              number
              url
              headRepository {
                nameWithOwner
              }
              headRefName
            }
          }
        }
      }
    }
  GRAPHQL
  jq = '.data.search.edges.[0].node.url'

  IO.popen(%W[gh api graphql -f q=#{query} -f query=#{graphql} --jq=#{jq}]) { |io|
    io.read.chomp[%r{\Ahttps://.*}]
  }
end

DOCSET_NAME = 'Trino'
DOCSET = "#{DOCSET_NAME.tr(' ', '_')}.docset"
DOCSET_ARCHIVE = File.basename(DOCSET, '.docset') + '.tgz'
ROOT_RELPATH = 'Contents/Resources/Documents'
INDEX_RELPATH = 'Contents/Resources/docSet.dsidx'
DOCS_ROOT = File.join(DOCSET, ROOT_RELPATH)
DOCS_INDEX = File.join(DOCSET, INDEX_RELPATH)
DOCS_URI = URI("https://trino.io/docs/#{ENV['BUILD_VERSION'] || 'current'}/")
DOCS_DIR = Pathname(DOCS_URI.host + DOCS_URI.path.chomp('/'))
ICON_URL = URI('https://avatars.githubusercontent.com/u/34147222?v=4&s=64')
ICON_FILE = Pathname('icon.png')
FETCH_LOG = 'wget.log'
DUC_REPO = 'Dash-User-Contributions'
DUC_OWNER = 'knu'
DUC_REMOTE_URL = "git@github.com:#{DUC_OWNER}/#{DUC_REPO}.git"
DUC_OWNER_UPSTREAM = 'Kapeli'
DUC_REMOTE_URL_UPSTREAM = "https://github.com/#{DUC_OWNER_UPSTREAM}/#{DUC_REPO}.git"
DUC_WORKDIR = DUC_REPO
DUC_DEFAULT_BRANCH = 'master'
DUC_BRANCH = 'trino'

def current_version
  ENV['BUILD_VERSION'] || extract_version()
end

def previous_version
  ENV['PREVIOUS_VERSION'] || Gem::Version.new(current_version).then { |current_version|
    Pathname.glob("versions/*/#{DOCSET}").map { |path|
      Gem::Version.new(path.parent.basename.to_s)
    }.select { |version|
      version < current_version
    }.max&.to_s
  }
end

def previous_docset
  version = previous_version or raise 'No previous version found'

  "versions/#{version}/#{DOCSET}"
end

def built_docset
  if version = ENV['BUILD_VERSION']
    "versions/#{version}/#{DOCSET}"
  else
    DOCSET
  end
end

def dump_index(docset, out)
  index = File.join(docset, INDEX_RELPATH)

  SQLite3::Database.new(index) do |db|
    db.execute("SELECT name, type, path FROM searchIndex ORDER BY name, type, path") do |row|
      out.puts row.join("\t")
    end
  end

  out.flush
end

def wget(*args)
  sh(
    'wget',
    '-nv',
    '--append-output', FETCH_LOG,
    '-N',
    '--retry-on-http-error=500,502,503,504',
    '-R', 'glossCatalog',
    *args
  )
end

desc "Fetch the #{DOCSET_NAME} document files."
task :fetch => %i[fetch:icon fetch:docs]

namespace :fetch do
  task :docs do
    puts 'Downloading %s' % DOCS_URI
    wget '-r', '--no-parent', '-p', DOCS_URI.to_s
  end

  task :icon do
    wget ICON_URL.to_s
    ln ICON_URL.route_from(ICON_URL + './').to_s, ICON_FILE, force: true
  end
end

file DOCS_DIR do
  Rake::Task[:'fetch:docs'].invoke
end

file ICON_FILE do
  Rake::Task[:'fetch:icon'].invoke
end

desc 'Build a docset in the current directory.'
task :build => [DOCS_DIR, ICON_FILE] do |t|
  rm_rf [DOCSET, DOCSET_ARCHIVE]

  mkdir_p DOCS_ROOT

  cp 'Info.plist', File.join(DOCSET, 'Contents')
  cp ICON_FILE, DOCSET

  cp_r DOCS_DIR.to_s + '/.', DOCS_ROOT

  # Index
  db = SQLite3::Database.new(DOCS_INDEX)

  db.execute(<<-SQL)
    CREATE TABLE searchIndex(id INTEGER PRIMARY KEY, name TEXT, type TEXT, path TEXT);
  SQL
  db.execute(<<-SQL)
    CREATE UNIQUE INDEX anchor ON searchIndex (name, type, path);
  SQL

  insert = db.prepare(<<-SQL)
    INSERT OR IGNORE INTO searchIndex(name, type, path) VALUES (?, ?, ?);
  SQL

  select = db.prepare(<<-SQL)
    SELECT TRUE FROM searchIndex where type = ? AND name = ?;
  SQL

  anchor_section = ->(path, node, name) {
    type = 'Section'
    a = Nokogiri::XML::Node.new('a', node.document)
    a['name'] = id = '//apple_ref/cpp/%s/%s' % [type, name].map { |s|
      URI.encode_www_form_component(s).gsub('+', '%20')
    }
    a['class'] = 'dashAnchor'
    node.prepend_child(a)
    # p [path, node.name, name]
  }

  index_item = ->(path, node, type, name) {
    if node
      a = Nokogiri::XML::Node.new('a', node.document)
      a['name'] = id = '//apple_ref/cpp/%s/%s' % [type, name].map { |s|
        URI.encode_www_form_component(s).gsub('+', '%20')
      }
      a['class'] = 'dashAnchor'
      node.prepend_child(a)
      url = "#{path}\##{id}"
    else
      url = path
    end
    insert.execute(name, type, url)
  }

  has_item = ->(type, name) {
    !select.execute!(type, name).empty?
  }

  version = extract_version or raise "Version unknown"

  puts "Generating docset for #{DOCSET_NAME} #{version}"

  cd DOCS_ROOT do
    File.open("_static/trino.css", "a") { |css|
      css.print <<~CSS

        /* Added for docset */
        header {
          display: none;
        }
        .md-main__inner, .md-container, .md-content__inner {
          padding-top: 0px;
        }
        dt:target {
          margin-top: 0px;
          padding-top: 0px;
        }
      CSS
    }

    Dir.glob('**/*.html') { |path|
      doc = Nokogiri::HTML(File.read(path), path)

      doc.css('link[href="https://fonts.gstatic.com/"][rel="preconnect"]').remove
      doc.css('link[href^="https://fonts.googleapis.com/"][rel="stylesheet"]').remove
      doc.css('link[href~="://"]').each { |node|
        warn "#{path} refers to an external resource: #{node.to_s}"
      }
      doc.css('#announcement').remove

      main = doc.at('article') or next

      if h1 = main.at_css('h1')
        index_item.(path, h1, 'Section', h1.text.chomp('#'))
      end
      main.css('h2, h3').each { |h|
        anchor_section.(path, h, h.text.chomp('#'))
      }

      case path
      when %r{\Afunctions/}
        case File.basename(path, '.html')
        when 'conditional'
          main.css('h2').each { |h|
            title = h.text.chomp('#')
            if h.at_xpath("./following-sibling::*[1]/code[string(.) = '#{title}' and starts-with(normalize-space(./following-sibling::text()[1]), 'expression ')]")
              case title
              when 'CASE', 'IF'
                index_item.(path, h, 'Query', title)
              else
                raise "Unknown expression: #{title}"
              end
            end
          }
        when 'decimal'
          main.css('.literal > .pre').each { |pre|
            case pre.text
            when 'DECIMAL'
              index_item.(path, pre, 'Operator', pre.text)
            end
          }

          main.at('.//h2[contains(translate(., "O", "o"), " operator")]/following-sibling::table').css('td:first-of-type .literal > .pre').each { |pre|
            case pre.text
            when %r{\A[+\-*/%]\z}
              index_item.(path, pre, 'Operator', pre.text)
            end
          }
          next
        when 'lambda'
          main.css('.literal > .pre').each { |pre|
            case pre.text
            when '->'
              index_item.(path, pre, 'Operator', pre.text)
            end
          }
        when 'list'
          next
        end

        main.xpath('.//h2[contains(translate(., "O", "o"), " operator")]').each { |h|
          h.xpath('./following-sibling::*').each { |el|
            case el.name
            when 'h1', 'h2'
              break
            when 'table'
              if el.at('.//th[string(.) = "Operator"]')
                el.css('td:first-of-type .literal > .pre').each_with_object({}) { |pre, seen|
                  op = pre.text
                  next if seen[op]

                  index_item.(path, pre, 'Operator', op)
                  seen[op] = true
                }
              end
            when 'p'
              el.css('.literal').each { |literal|
                case op = literal.text.gsub(/\s+/, ' ')
                when /\A(?:NULL|TRUE|FALSE)\z|\A[a-z]/
                  # ignore
                else
                  index_item.(path, literal, 'Operator', op)
                end
              }
            end
          }
        }

        main.css('h2').each { |h2|
          case h2.text.chomp('#')
          when /\A(?:[\w ]+: )?(?<operators>(?:(?:(?<op>(?:(?<w>[A-Z]+) )*\g<w>), )*\g<op> and )?\g<op>)\z/
            $~[:operators].scan(/(?<op>(?:(?<w>[A-Z]+) )*\g<w>)/) {
              index_item.(path, h2, 'Operator', $~[:op])
            }
          end
        }
      when %r{\Aconnector/}
        main.xpath('.//table[.//th[translate(., "NP", "np") = "property name"] and ./ancestor::section[contains(@id, "properties") or contains(@id, "configuration")]]').each do |table|
          table.css('td:first-of-type .literal > .pre').each do |pre|
            index_item.(path, pre, 'Variable', pre.text)
          end
        end

        connector_name = nil

        main.css('section#procedures > ul').each do |el|
          connector_name ||=
            main.xpath('//section[contains(@id, "configuration")]//pre').find { |pre|
              if name = pre.xpath('normalize-space(.)')[/^connector.name=\K\w+/]
                break name
              end
            } or raise "#{path}: connector.name not found"

          el.xpath('./li/p[position() = 1]/code[position() = 1]').each do |para|
            if procedure = para.xpath('normalize-space(.)')[/\A(?:CALL\s+)?\K[^(]+/]
              if procedure.delete_prefix!('example.')
                warn "#{path}: example. prefix is found and removed: #{procedure}"
              end
              index_item.(path, el, 'Procedure', "#{connector_name}.#{procedure}")
            end
          end
        end
      when 'language/types.html'
        main.css('h3 > code').each { |code|
          case text = code.text.chomp('#')
          when /\A(?<w>[A-Z][A-Za-z0-9]*(\(P\))?)( \g<w>)*\z/
            index_item.(path, code.parent, 'Type', text)
          else
            raise "Unknown type name: #{text}"
          end
        }
      when %r{\Asql/}
        main.css('h1').each { |h1|
          case h1.text.chomp('#')
          when /\A(?<st>((?<w>[A-Z]+) )*\g<w>)\z/
            index_item.(path, h1, 'Statement', $~[:st])
          end
        }
        if path == 'sql/select.html'
          main.css('h2, h3').each { |h|
            case h.text.chomp('#')
            when /\A(?<queries>(?:(?<q>(?:(?<w>[A-Z]+) )*\g<w>) or )*\g<q>)(?: clause)?\z/
              $~[:queries].scan(/(?<q>(?:(?<w>[A-Z]+) )*\g<w>)/) {
                index_item.(path, h, 'Query', $~[:q])
              }
            end
          }
        end
      when 'glossary.html'
        main.css('a[href^="gloss"]:not([href$=".html"])').each { |a|
          href = a['href']
          warn "bad link in #{path}: #{href}"
          a['href'] = '#' + href.downcase
        }
      end

      main.css('.descname').each { |descname|
        func =
          if (prev = descname.previous).matches?('.descclassname')
            prev.text + descname.text
          else
            descname.text
          end
        type =
          if descname.at_xpath('./ancestor::section[contains(@id, "procedures")]')
            'Procedure'
          else
            'Function'
          end
        index_item.(path, descname, type, func)
      }

      File.write(path, doc.to_s)
    }
  end

  select.close
  insert.close

  get_count = ->(**criteria) do
    db.get_first_value(<<-SQL, criteria.values)
      SELECT COUNT(*) from searchIndex where #{
        criteria.each_key.map { |column| "#{column} = ?" }.join(' and ')
      }
    SQL
  end

  assert_exists = ->(**criteria) do
    if get_count.(**criteria).zero?
      raise "#{criteria.inspect} not found in index!"
    end
  end

  puts 'Performing sanity check'

  {
    'Statement' => ['SELECT', 'INSERT', 'UPDATE', 'DELETE',
                    'REFRESH MATERIALIZED VIEW'],
    'Query' => ['CROSS JOIN',
                'UNION', 'INTERSECT', 'EXCEPT',
                'GROUP BY', 'LIMIT', 'FETCH FIRST', 'OFFSET',
                'IN', 'EXISTS', 'UNNEST',
                'IF', 'CASE'],
    'Function' => ['count', 'merge',
                   'array_sort', 'rank',
                   'if', 'coalesce', 'nullif'],
    'Procedure' => ['runtime.kill_query',
                    'kudu.system.add_range_partition',
                    'kudu.system.drop_range_partition',
                    'hive.system.create_empty_partition',
                    'hive.system.drop_stats',
                    'postgresql.system.flush_metadata_cache'],
    'Type' => ['BOOLEAN', 'BIGINT', 'DOUBLE', 'DECIMAL', 'VARCHAR', 'VARBINARY',
               'DATE', 'TIMESTAMP', 'TIMESTAMP(P) WITH TIME ZONE',
               'ARRAY', 'UUID', 'HyperLogLog'],
    'Operator' => ['+', '<=', '!=', '<>', '[]', '||',
                   'BETWEEN', 'NOT BETWEEN', 'LIKE', 'AND', 'OR', 'NOT',
                   'ANY', 'IS NULL', 'IS DISTINCT FROM'],
    'Variable' => ['pinot.connection-timeout', 'trino.thrift.client.connect-timeout'],
    'Section' => ['BigQuery connector']
  }.each { |type, names|
    names.each { |name|
      assert_exists.(name: name, type: type)
    }
  }

  db.close

  sh 'tar', '-zcf', DOCSET_ARCHIVE, '--exclude=.DS_Store', DOCSET

  mkdir_p "versions/#{version}/#{DOCSET}"
  sh 'rsync', '-a', '--exclude=.DS_Store', '--delete', "#{DOCSET}/", "versions/#{version}/#{DOCSET}/"

  puts "Finished creating #{DOCSET} #{version}"

  system paginate_command('rake diff:index', diff: true)
end

task :dump do
  system paginate_command('rake dump:index')
end

namespace :dump do
  desc 'Dump the index.'
  task :index do
    dump_index(built_docset, $stdout)
  end
end

task :diff do
  system paginate_command('rake diff:index diff:docs', diff: true)
end

namespace :diff do
  desc 'Show the differences in the index from an installed version.'
  task :index do
    Tempfile.create(['old', '.txt']) do |otxt|
      Tempfile.create(['new', '.txt']) do |ntxt|
        dump_index(previous_docset, otxt)
        otxt.close
        dump_index(built_docset, ntxt)
        ntxt.close

        puts "Diff in document indexes:"
        sh 'diff', '-U3', otxt.path, ntxt.path do
          # ignore status
        end
      end
    end
  end

  desc 'Show the differences in the docs from an installed version.'
  task :docs do
    old_root = File.join(previous_docset, ROOT_RELPATH)
    new_root = File.join(built_docset, ROOT_RELPATH)

    puts "Diff in document files:"
    sh 'diff', '-rwNU3',
      '-x', '*.js',
      '-x', '*.css',
      '-x', '*.svg',
      '-I', '^[[:space:]]+VERSION:[[:space:]]+\'[0-9.]+\',',
      '-I', 'Trino [0-9]+',
      '-I', '^[[:space:]]*$',
      old_root, new_root do
      # ignore status
    end
  end
end

file DUC_WORKDIR do |t|
  sh 'git', 'clone', DUC_REMOTE_URL, t.name
  cd t.name do
    sh 'git', 'remote', 'add', 'upstream', DUC_REMOTE_URL_UPSTREAM
    sh 'git', 'remote', 'update', 'upstream'
  end
end

desc 'Push the generated docset if there is an update'
task :push => DUC_WORKDIR do
  version = extract_version
  workdir = Pathname(DUC_WORKDIR) / 'docsets' / File.basename(DOCSET, '.docset')

  docset_json = workdir / 'docset.json'
  archive = workdir / DOCSET_ARCHIVE
  versioned_archive = workdir / 'versions' / version / DOCSET_ARCHIVE

  puts "Resetting the working directory"
  cd workdir.to_s do
    sh 'git', 'remote', 'update'

    start_ref =
      if git_ref_exists?("origin/#{DUC_BRANCH}")
        "origin/#{DUC_BRANCH}"
      else
        "upstream/#{DUC_DEFAULT_BRANCH}"
      end
    if git_ref_exists?(DUC_BRANCH)
      sh 'git', 'checkout', DUC_BRANCH
      sh 'git', 'reset', '--hard', start_ref
    else
      sh 'git', 'checkout', '-b', DUC_BRANCH, start_ref
    end

    base_versions, head_versions = %W[upstream/#{DUC_DEFAULT_BRANCH} HEAD].map { |branch|
      JSON.parse(
        capture(*%W[git cat-file -p #{branch}:#{docset_json.relative_path_from(DUC_WORKDIR)}])
      )['specific_versions'].map { |o| o['version'] }
    }
    if (head_versions - base_versions - [version]).empty?
      sh 'git', 'reset', '--hard', "upstream/#{DUC_DEFAULT_BRANCH}"
    end
  end

  cp DOCSET_ARCHIVE, archive
  mkdir_p versioned_archive.dirname
  cp archive, versioned_archive

  puts "Updating #{docset_json}"
  updated = false
  File.open(docset_json, 'r+') { |f|
    json = JSON.parse(f.read)
    json['specific_versions'] = specific_versions = json['specific_versions'].to_set.add(
      {
        'version' => version,
        'archive' => versioned_archive.relative_path_from(workdir).to_s
      }
    ).sort_by { |o| Gem::Version.new(o['version']) }.reverse
    latest_version = specific_versions.dig(0, 'version')
    updated = json['version'] != latest_version
    json['version'] = latest_version
    f.rewind
    f.puts JSON.pretty_generate(json, indent: "    ")
    f.truncate(f.tell)
  }

  cd workdir.to_s do
    json_path = docset_json.relative_path_from(workdir).to_s

    if system(*%W[git diff --exit-code --quiet #{json_path}])
      puts "Nothing to commit."
      next
    end

    sh paginate_command(%W[git diff #{json_path}], diff: true)

    sh(*%W[git add], *[archive, versioned_archive, docset_json].map { |path| path.relative_path_from(workdir).to_s })
    message =
      if updated
        "Update #{DOCSET_NAME} docset to #{version}"
      else
        "Add #{DOCSET_NAME} docset #{version}"
      end
    sh(*%W[git commit -m #{message}])
    sh(*%W[git push -fu origin #{DUC_BRANCH}:#{DUC_BRANCH}])

    if pull_url = existing_pull_url
      puts "Updating the title of the existing pull-request: #{pull_url}"
      sh(*%W[gh pr edit #{pull_url} --title #{message}])
    end
  end

  unless `git tag -l v#{version}`.empty?
    sh(*%W[git tag -d v#{version}]) {}
    sh(*%W[gh release delete --cleanup-tag --yes v#{version}]) {}
  end

  sh(*%W[gh release create v#{version} -t v#{version} -n #{<<~NOTES} -- #{archive}])
    This docset is generated from the official documentation of Trino.

    #{DOCS_URI}

    Trino is open source software licensed under the [Apache License 2.0](https://github.com/trinodb/trino/blob/master/LICENSE) and supported by the [Trino Software Foundation](https://trino.io/foundation). See trademark and other [legal notices](https://trino.io/legal).
  NOTES

  puts "New docset is committed and pushed to #{DUC_OWNER}:#{DUC_BRANCH}.  To send a PR, go to the following URL:"
  puts "\t" + "#{DUC_REMOTE_URL_UPSTREAM.delete_suffix(".git")}/compare/#{DUC_DEFAULT_BRANCH}...#{DUC_OWNER}:#{DUC_BRANCH}?expand=1"
end

desc 'Send a pull-request'
task :pr => DUC_WORKDIR do
  cd DUC_WORKDIR do
    sh(*%W[git diff --exit-code --stat #{DUC_BRANCH}..upstream/#{DUC_DEFAULT_BRANCH}]) do |ok, _res|
      if ok
        puts "Nothing to send a pull-request for."
      else
        sh(*%W[gh repo set-default #{DUC_OWNER_UPSTREAM}/#{DUC_REPO}])
        sh(*%W[
          gh pr create
          --base #{DUC_DEFAULT_BRANCH}
          --head #{DUC_OWNER}:#{DUC_REPO}:#{DUC_BRANCH}
          --title #{capture(*%W[git log -1 --pretty=%s #{DUC_BRANCH}]).chomp}
        ])
      end
    end
  end
end

desc 'Delete all fetched files and generated files'
task :clean do
  rm_rf [DOCS_DIR, ICON_FILE, DOCSET, DOCSET_ARCHIVE, FETCH_LOG]
end

task :default => :build
