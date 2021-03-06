ssh-agent-filter
================


the problem we want to solve
----------------------------

ssh(1) describes the problem:
> Agent forwarding should be enabled with caution.  Users with the ability to bypass file permissions on the remote host (for the agent's UNIX-domain socket) can access the local agent through the forwarded connection.  An attacker cannot obtain key material from the agent, however they can perform operations on the keys that enable them to authenticate using the identities loaded into the agent.


our solution
------------

1. create one key(pair) for each realm you connect to
2. load keys into your `ssh-agent` as usual
3. use `ssh-agent-filter` to allow only the key(s) you need

`afssh` (agent filtered ssh) can wrap `ssh-agent-filter` and `ssh` for you, forwarding only the key with the comment `id_example`:

    $ afssh --comment id_example -- example.com

starts an `ssh-agent-filter --comment id_example`, runs `ssh -A example.com` and kills the `ssh-agent-filter` afterwards.

If you leave out the options before the `--`:

    $ afssh -- example.com

it will ask you via `whiptail` or `dialog` which keys you want to have forwarded.


confirmation
------------

You can use the `--*-confirmed` options (e.g.`--comment-confirmed`) to add keys for which you want to be asked on each use through the filter.
The confirmation is done in the same way as when you `ssh-add -c` a key to your `ssh-agent`, but the question will contain some additional information extracted from the sign request.


how it works
------------

ssh-agent-filter provides a socket interface identical to that of a normal ssh-agent.
We don't keep private key material, but delegate requests to the upstream ssh-agent after checking if the key is allowed.

The following requests are implemented:
* `SSH2_AGENTC_REQUEST_IDENTITIES`:
  * asks for a list of SSH 2 keys
  * the upstream ssh-agent is asked for that list and the result is filtered
* `SSH2_AGENTC_SIGN_REQUEST`:
  * asks for a signature on some data to be made with a key
  * if the key is allowed the request is forwarded to the upstream ssh-agent and the result returned
  * else failure is returned
* `SSH_AGENTC_REQUEST_RSA_IDENTITIES`:
  * asks for a list of SSH 1 keys
  * an empty list is returned
* `SSH_AGENTC_REMOVE_ALL_RSA_IDENTITIES`:
  * asks for removal of all SSH 1 keys
  * success is returned without doing anything

Requests to add or remove keys and to (un)lock the agent are refused
