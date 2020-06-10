# Oracle encoding error

sqlplus 실행 시, `???`로 표시되는 등 인코딩이 깨졌을 때 NLS_LANG 값 수정:

```bash
$ export NLS_LANG=AMERICAN_AMERICA.KO16KSC5601
```