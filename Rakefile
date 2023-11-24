# Requirements
# * ruby
# * curl
# * gh
#   * https://github.com/cli/cli
#   * gh auth login
#
# Usage
# * rake

# デプロイ先のサーバ
HOSTS = {
  host01: "isu10f1", # 43.207.98.210", # xxx.xxx.xxx.xxx", # front, db, app(uploader, auth), redis, memcached
  host02: "isu10f2", # 18.179.23.80", # xxx.xxx.xxx.xxx",  # app(main)
  host03: "isu10f3", # 43.207.113.41", # xxx.xxx.xxx.xxx", # app(main)
}

def bench_command(timestamp)
<<~EOS
sudo systemd-run \
  --working-directory=/home/isucon/benchmarker \
  --pipe \
  --wait \
  --collect \
  --uid=$(id -u)\
  --gid=$(id -g) \
  --slice=benchmarker.slice \
  --service-type=oneshot \
  -p AmbientCapabilities=CAP_NET_BIND_SERVICE \
  -p CapabilityBoundingSet=CAP_NET_BIND_SERVICE \
  -p LimitNOFILE=2000000 \
  -p TimeoutStartSec=110s \
    ~isucon/benchmarker/bin/benchmarker \
    -exit-status \
    -tls \
    -target app.t.isucon.dev \
    -host-advertise bench.t.isucon.dev \
    -push-service-port 1001 \
    -tls-cert /etc/ssl/private/tls-cert.pem \
    -tls-key /etc/ssl/private/tls-key.pem | tee /tmp/bench/#{timestamp}.txt
EOS
end

BENCH_IP = "isu10fb"
# INITIALIZE_ENDPOINT = "https://isucondition.t.isucon.dev/initialize_from_local"

# デプロイ先のカレントディレクトリ
CURRENT_DIR = "/home/isucon/webapp"

# rubyアプリのディレクトリ
RUBY_APP_DIR = "/home/isucon/webapp/ruby"

# アプリのservice名
APP_SERVICE_API_NAME = "xsuportal-api-ruby.service"
APP_SERVICE_WEB_NAME = "xsuportal-web-ruby.service"

# デプロイを記録するissue
GITHUB_REPO     = "kenzan100/isuconNov24-practice" # sue445/isucon11-qualify"
GITHUB_ISSUE_ID = 1

BUNDLE = "/home/isucon/local/ruby/bin/bundle"

ALP_MATCHING_GROUP = [
  "/api/condition/[0-9a-z\-]+$",
  "/api/condition/[0-9a-z\-]/icon$",
  "/api/condition/[0-9a-z\-]/graph$",
  "/api/isu/[0-9a-z\-]+$",
  "/api/isu/[0-9a-z\-]+/icon$",
  "/api/isu/[0-9a-z\-]+/graph$",
  "/isu/[0-9a-z\-]+/condition$",
  "/isu/[0-9a-z\-]+/graph$",
  "/isu/[0-9a-z\-]+$"
].join(',')

def exec(ip_address, command, cwd: CURRENT_DIR)
  sh %Q(ssh isucon@#{ip_address} 'cd #{cwd} && #{command}')
end

namespace :deploy do
  HOSTS.each do |name, ip_address|
    desc "Deploy to #{name}"
    task name do
      puts "[deploy:#{name}] START"

      # common
      exec ip_address, "git pull origin main --ff"

      # systemdの更新後にdaemon-reloadする
      exec ip_address, "sudo systemctl daemon-reload"

      # TODO: 終了10分前にdisableすること！！！！！！
      # exec ip_address, "sudo systemctl restart newrelic-infra"
      # exec ip_address, "sudo systemctl disable newrelic-infra"
      # exec ip_address, "sudo systemctl stop newrelic-infra"
      # exec ip_address, "sudo systemctl enable newrelic-infra"
      # exec ip_address, "sudo systemctl start newrelic-infra"

      # mysql
      case name
      when :host01
        # exec ip_address, "sudo cp infra/mysql/slowquery.cnf /etc/mysql/mysql.conf.d/slowquery.cnf"
        exec ip_address, "sudo mysqld --verbose --help > /dev/null"
        exec ip_address, "echo -n | sudo tee /var/log/mysql/slow.log"
        exec ip_address, "sudo systemctl restart mysql"
      else
        exec ip_address, "sudo systemctl stop mysql"
      end

      # nginx
      case name
      when :host01
        exec ip_address, "sudo systemctl disable --now envoy"
        exec ip_address, "sudo systemctl stop envoy"

        exec ip_address, "sudo cp infra/nginx/nginx.conf  /etc/nginx/sites-enabled/isucondition.conf"
        exec ip_address, "sudo nginx -t"
        exec ip_address, "sudo rm -f /home/isucon/access.log"
        exec ip_address, "sudo systemctl restart nginx"
      else
        exec ip_address, "sudo systemctl disable --now envoy"
        exec ip_address, "sudo systemctl stop envoy"
        exec ip_address, "sudo systemctl stop nginx"
      end

      # app
      case name
      when :host01, :host02, :host03
        exec ip_address, "#{BUNDLE} install --path vendor/bundle --jobs $(nproc)", cwd: RUBY_APP_DIR
        exec ip_address, "#{BUNDLE} config set --local path 'vendor/bundle'", cwd: RUBY_APP_DIR
        exec ip_address, "#{BUNDLE} config set --local jobs $(nproc)", cwd: RUBY_APP_DIR
        exec ip_address, "#{BUNDLE} install", cwd: RUBY_APP_DIR

        # exec ip_address, "sudo systemctl disable --now isuxportal-web-golang.service"
        # exec ip_address, "sudo systemctl stop isuxportal-web-golang.service"

        exec ip_address, "sudo systemctl stop #{APP_SERVICE_API_NAME}"
        exec ip_address, "sudo systemctl stop #{APP_SERVICE_WEB_NAME}"
        exec ip_address, "sudo systemctl start #{APP_SERVICE_API_NAME}"
        exec ip_address, "sudo systemctl start #{APP_SERVICE_WEB_NAME}"
        exec ip_address, "sudo systemctl status #{APP_SERVICE_API_NAME}"
        exec ip_address, "sudo systemctl status #{APP_SERVICE_WEB_NAME}"
      else
        exec ip_address, "sudo systemctl stop #{APP_SERVICE_API_NAME}"
        exec ip_address, "sudo systemctl stop #{APP_SERVICE_WEB_NAME}"
      end

      exec ip_address, "sudo rm -f /tmp/sql.log"
      exec ip_address, "rm -rf tmp/stackprof/*", cwd: RUBY_APP_DIR

      # memcached
      # case name
      # when :host01
      #   exec ip_address, "sudo cp infra/memcached/memcached.conf /etc/memcached.conf"
      #   exec ip_address, "sudo systemctl restart memcached"
      # else
      #   exec ip_address, "sudo systemctl stop memcached"
      # end

      # redis
      # case name
      # when :host01
      #   exec ip_address, "sudo cp infra/redis/redis.conf /etc/redis/redis.conf"
      #   exec ip_address, "sudo systemctl restart redis-server"
      # else
      #   exec ip_address, "sudo systemctl stop redis-server"
      # end

      # sidekiq
      # case name
      # when :host01
      #   # exec ip_address, "#{BUNDLE} install --path vendor/bundle --jobs $(nproc)", cwd: "#{CURRENT_DIR}/webapp/ruby"
      #   # exec ip_address, "sudo systemctl stop isutrain-sidekiq.service"
      #   # exec ip_address, "sudo systemctl start isutrain-sidekiq.service"
      #   # exec ip_address, "sudo systemctl status isutrain-sidekiq.service"
      # else
      #   # exec ip_address, "sudo systemctl stop isutrain-sidekiq.service"
      # end

      # docker-compose
      # case name
      # when :host01
      #   # exec ip_address, "docker-compose -f webapp/docker-compose.yml -f webapp/docker-compose.ruby.yml down"
      #   # exec ip_address, "docker-compose -f webapp/docker-compose.yml up -d --build"
      # else
      #   # exec ip_address, "docker-compose -f webapp/docker-compose.yml -f webapp/docker-compose.ruby.yml down"
      # end

      puts "[deploy:#{name}] END"
    end
  end
end

desc "Prepare for deploy"
task :setup do
  sh "git push"
end

desc "Deploy to all hosts"
multitask :deploy => HOSTS.keys.map { |name| "deploy:#{name}" }

desc "POST /initialize"
task :initialize do
  # sh "curl --max-time 20 -X POST --retry 3 --fail #{INITIALIZE_ENDPOINT}"
end

desc "Record current commit to issue"
task :record do
  revision = `git rev-parse --short HEAD`.strip

  current_tag = [
    Time.now.strftime("%Y%m%d-%H%M%S"),
    `whoami`.strip
  ].join("-")

  message = ":rocket: Deployed #{revision} [#{current_tag}](https://github.com/#{GITHUB_REPO}/releases/tag/#{current_tag})"

  # 直前のリリースのtagを取得する
  before_tag = `git tag | tail -n 1`.strip

  unless before_tag.empty?
    message << " ([compare](https://github.com/#{GITHUB_REPO}/compare/#{before_tag}...#{current_tag}))"
  end

  sh "git tag -a #{current_tag} -m 'Release #{current_tag}'"
  sh "git push --tags"

  sh "gh issue comment --repo #{GITHUB_REPO} #{GITHUB_ISSUE_ID} --body '#{message}'"
end

task :all => [:setup, :deploy, :initialize, :record]

task :default => :all

desc "alp_install"
task :alp_install do
  HOSTS.each do |name, ip_address|

    exec ip_address, "sudo apt-get install unzip"
    exec ip_address, "wget https://github.com/tkuchiki/alp/releases/download/v1.0.21/alp_linux_amd64.zip"
    exec ip_address, "unzip alp_linux_amd64.zip"
    exec ip_address, "sudo install alp /usr/local/bin/alp"
  end
end

desc "slp_install"
task :slp_install do
  HOSTS.each do |name, ip_address|
    exec ip_address, "wget https://github.com/tkuchiki/slp/releases/download/v0.2.0/slp_linux_amd64.zip"
    exec ip_address, "unzip slp_linux_amd64.zip"
    exec ip_address, "sudo install slp /usr/local/bin/slp"
  end
end

desc "bench"
task :bench do
#   exec BENCH_IP, "sudo systemctl stop jiaapi-mock.service"
  timestamp = Time.now.strftime('%Y%m%d%H%M')
  exec BENCH_IP, "mkdir -p /tmp/bench"
  exec BENCH_IP, bench_command(timestamp), cwd: "/home/isucon/benchmarker"
  sh "scp #{BENCH_IP}:/tmp/bench/#{timestamp}.txt ./log/bench/#{timestamp}.txt"

  exec HOSTS[:host01], "mkdir -p /tmp/alp"
  exec HOSTS[:host01], "mkdir -p /tmp/slp"

  exec HOSTS[:host01], "alp ltsv --file=/home/isucon/access.log -r --sort=sum -m '#{ALP_MATCHING_GROUP}' --format html > /tmp/alp/#{timestamp}.html"
  sh "scp #{HOSTS[:host01]}:/tmp/alp/#{timestamp}.html ./log/alp/#{timestamp}.html"

  # exec HOSTS[:host01], "sudo cat /var/log/mysql/slow.log | slp my --format html > /tmp/slp/#{timestamp}.html"
  # sh "scp #{HOSTS[:host01]}:/tmp/slp/#{timestamp}.html ./log/slp/#{timestamp}.html"

  sh "git pull"
  sh "git add -A"
  sh "git commit -m 'bench #{timestamp}'"
  sh "git push origin main"
end
