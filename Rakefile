require 'open3'

RED = "\033[31m".freeze
GREEN = "\033[32m".freeze
RESET = "\033[0m".freeze

# Let's ignore rubocop here until such a time as we either muzzle it
# or get these helpers more centralized and tested.
# rubocop:disable Metrics/AbcSize,Metrics/CyclomaticComplexity,Metrics/MethodLength
def run_command(cmd, silent: true, print_command: false, report_status: false)
  cmd_string = cmd.is_a?(String) ? cmd : cmd.join(' ')
  puts "#{GREEN}Running #{cmd_string}#{RESET}" if print_command
  output = ''
  Open3.popen2e(*Array(cmd)) do |_stdin, stdout_stderr, thread|
    stdout_stderr.each do |line|
      puts line unless silent
      output += line
    end
    exitcode = thread.value.exitstatus
    unless exitcode.zero?
      err = "#{RED}Command failed! Command: #{cmd_string}, Exit code: #{exitcode}"
      # Print details if we were running silent
      err += "\nOutput:\n#{output}" if silent
      err += RESET
      abort err
    end
    puts "#{GREEN}Command finished with status #{exitcode}#{RESET}" if report_status
  end
  output.chomp
end
# rubocop:enable Metrics/AbcSize,Metrics/CyclomaticComplexity,Metrics/MethodLength

Dir.glob(File.join('tasks/**/*.rake')).each { |file| load file }

### Puppetlabs stuff ###
require 'packaging'

load './ext/release-lead.rake'

Pkg::Util::RakeUtils.load_packaging_tasks

desc 'run static analysis with rubocop'
task(:rubocop) do
  require 'rubocop'
  cli = RuboCop::CLI.new
  exit_code = cli.run(%w(--display-cop-names --format simple))
  raise "RuboCop detected offenses" if exit_code != 0
end

desc "verify that commit messages match CONTRIBUTING.md requirements"
task(:commits) do
  # This rake task looks at the summary from every commit from this branch not
  # in the branch targeted for a PR. This is accomplished by using the
  # TRAVIS_COMMIT_RANGE environment variable, which is present in travis CI and
  # populated with the range of commits the PR contains. If not available, this
  # falls back to `main..HEAD` as a next best bet as `main` is unlikely to
  # ever be absent.
  #
  # When we move to GH actions, use `GITHUB_BASE_REF` to resolve the merge base
  # ref, which is the common ancestor between the base branch and PR. Then do
  # git log for all of the commits in `HEAD` that are not in the base ref
  #
  #   baseref = %x{git merge-base HEAD $GITHUB_BASE_REF}
  #   commits = "#{baseref}..HEAD"
  commits = ENV['TRAVIS_COMMIT_RANGE'].nil? ? 'main..HEAD' : ENV['TRAVIS_COMMIT_RANGE'].sub(/\.\.\./, '..')
  %x{git log --no-merges --pretty=%s #{commits}}.each_line do |commit_summary|
    error_message=<<-HEREDOC
\n\n\n\tThis commit summary didn't match CONTRIBUTING.md guidelines:\n \
\n\t\t#{commit_summary}\n \
\tThe commit summary (i.e. the first line of the commit message) should start with one of:\n  \
\t\t(docs)\n \
\t\t(maint)\n \
\t\t(packaging)\n \
\t\t(<ANY PUBLIC JIRA TICKET>)\n \
\n\tThis test for the commit summary is case-insensitive.\n\n\n
    HEREDOC

    if /^\((maint|doc|docs|packaging|pa-\d+)\)|revert|bumping|merge|promoting/i.match(commit_summary).nil?
      ticket = commit_summary.match(/^\(([[:alpha:]]+-[[:digit:]]+)\).*/)
      if ticket.nil?
        raise error_message
      else
        require 'net/http'
        require 'uri'
        uri = URI.parse("https://tickets.puppetlabs.com/browse/#{ticket[1]}")
        response = Net::HTTP.get_response(uri)
        if response.code != "200"
          raise error_message
        end
      end
    end
  end
end
