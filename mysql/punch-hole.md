# Punch hole

[포스트](https://mysqlserverteam.com/innodb-transparent-page-compression/) 참고해 정리

## 페이지 압축이 동작하는 방식

페이지가 쓰여질 때, 해당 페이지는 지정된 압축 알고리즘을 사용하여 압축된다. 압축된 데이터는 디스크에 기록된다. 여기서 hole punching 메커니즘은 페이지 끝에서 빈 블록을 해제한다. 압축에 실패하면, 데이터는 있는 그대로 쓰여진다.

## Linux 상에서의 Hole Punch 크기

Linux 시스템에서 파일 시스템의 블록 크기는 hole punching에 사용되는 단위 크기다. 따라서, 페이지 압축은 `InnoDB 페이지 크기 - 파일 시스템 블록 크기`보다 작거나 같은 크기로 압축될 수 있는 경우에만 작동한다. 예를 들어, [innodb_page_size](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_page_size)가 16KB고 파일 시스템 블록 크기가 4KB인 경우, hole punching을 가능하게 하려면 페이지 데이터를 12KB 이하로 압축해야 한다.
