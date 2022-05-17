### WMIとは？

> WMI（Windows Management Instrumentation）は、Windows OSを管理することを目的にMicrosoftが開発した技術です。WMIを活用することで、Windowsシステムの状態を示す情報　
> （CPU使用率・Windowsサービスの起動状態、起動しているプロセス、発生したログ・アプリケーションログetc.）を取得できます。

# monitor関数について

```python
def monitor():
    log_to_file('CommandLine, Time, Executable, Parent PID, PID, User, Privileges')
    c = wmi.WMI()
    process_watcher = c.Win32_Process.watch_for('creation')
    while True:
        try:
            new_process = process_watcher()
            cmdline = new_process.CommandLine
            create_date = new_process.CreationDate
            executable = new_process.ExecutablePath
            parent_pid = new_process.ParentProcessId
            pid = new_process.ProcessId
            proc_owner = new_process.GetOwner()
            
            privileges = 'N/A'
            process_log_message = (
                f'{cmdline} , {create_date} , {executable},'
                f'{parent_pid} , {pid} , {proc_owner} , {privileges}'
                )
            print(process_log_message)
            print()
            log_to_file(process_log_message)
        except Exception:
            pass
```

まずWMIクラスのインスタンスを作ってローカルマシーンへ接続。

`Win32_Process.watch_for('creation')`で、プロセス生成のイベントを監視するように設定。
 
`new_process`はWin32_Processと呼ばれるWMIクラスのインスタン。

Win32_Processには上記のような様々な情報が格納されている。

詳しくはこちら：[Win32_Process クラス](https://docs.microsoft.com/ja-jp/windows/win32/cimwin32prov/win32-process)

### トークンとは？

> 各プロセスは、それぞれ自分自身のトークンを持っている。トークン（token）とは、運転免許証に例えることができる。トークンには、そのトークンの持ち主が誰であるかや
> （免許証でいえば氏名など）、どのような権利を持っているか（免許証でいえば普通自動車や自動二輪の運転が可能である、など）が記載されている。なお、ここではトークンを
> IDカードやパスポートではなく（これらには、通常は名前情報だけが記入されている）、免許証に例えている理由は、トークンには名前だけではなく、「～する権利」も含まれて
> いるからである。Windows OSは、アクセスしようとしているプロセスのトークンと、アクセスされるオブジェクトのACLを比較し、オブジェクトへのアクセスを認めるかどうかを判定する。

# get_process_privileges関数について

```python
def get_process_privileges(pid):
    try:
        hproc = win32api.OpenProcess(win32con.PROCESS_QUERY_INFORMATION, False, pid)
        htok = win32security.OpenProcessToken(hproc, win32con.TOKEN_QUERY)
        privs = win32security.GetTokenInformation(htok, win32security.TokenPrivileges)
        privileges = ''
        for priv_id, flags in privs:
            if flags == win32security.SE_PRIVILEGE_ENABLED | win32security.SE_PRIVILEGE_ENABLED_BY_DEFAULT:
                privileges += f'{win32security.LookupPrivilegeName(None, priv_id)}|'
    except Exception:
        privileges = 'N/A'

    return privileges
```

まず`OpenProcess`でターゲットプロセスへのハンドルを取得する。

引数は

- プロセスへのアクセスに関するオプション
- プロセス作成時に取得したハンドルを継承するか否か
- プロセスID

第一引数の`PROCESS_QUERY_INFORMATION`は、取得したハンドルを`GetExitCodeProcess`関数、及び`GetPriorityClass`関数で使用できるようするという意味。

次に`OpenProcessToken`でプロセスに関連付けられたトークンを開く。

引数は

- プロセスのハンドル
- プロセスに対するアクセス権

`GetTokenInformation`でトークンに関する情報を取得。

引数は

- トークンへのハンドル
- トークン情報クラス

第二引数は、関数が取得する情報のタイプについてのもの。

ここでは、`privs`は”タプルのリスト”（後のforループに注目）。

0番目が特権（privilege）、1番目がその特権がenableか否か。

> ((int,int),) returns PyTOKEN_PRIVILEGES (tuple of LUID and attribute flags for each privilege) attributes are combination of SE_PRIVILEGE_ENABLED,
> SE_PRIVILEGE_ENABLED_BY_DEFAULT,SE_PRIVILEGE_USED_FOR_ACCESS

- SE_PRIVILEGE_ENABLED…The privilege is enabled.
- SE_PRIVILEGE_ENABLED_BY_DEFAULT…The privilege is enabled by default.

ifの条件は、特権がenabledになっているかの確認をしてる。

`LookupPrivilegeName`で特権名を得る。
