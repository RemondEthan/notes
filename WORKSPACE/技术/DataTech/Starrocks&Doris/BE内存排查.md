```bash
curl -XGET -s http://ip:8040/metrics | grep "^starrocks_be_.*_mem_bytes"
```
```
curl http://be_ip:8040/mem_tracker 
```
```
1.admin execute on {beid} 'System.print(HeapProf.getInstance().enable_prof())';
2.admin execute on {beid} 'System.print(HeapProf.getInstance().has_enable())';
3.admin execute on {beid} 'System.print(HeapProf.getInstance().dump_dot_snapshot())';
把3的结果给我发下然后
4.admin execute on {beid} 'System.print(HeapProf.getInstance().disable_prof())';
```