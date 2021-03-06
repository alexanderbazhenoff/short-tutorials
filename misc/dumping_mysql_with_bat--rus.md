# Backing up with batch command line

Когда-то мне нужно было периодически создавать резервные копии MySQL и пользовательских данных с помощью обычного bat. Качаем [comandline-верисю 7za.exe](https://medium.com/r/?url=http%3A%2F%2Fwww.7-zip.org%2Fdownload.html) и сохраняем в ту же папку, где и будет расположен batch comand sctipt. Этот bat сохраняет dump MySQL, запаковывает в 7zip и копирует на заранее смонтированный сетевой диск:

```bat
title MySQL Dump batch file
set ctime=%TIME:~0,2%
if "%ctime:~0,1%" == " " (set ctime=0%ctime:~1,1%) 
set ctime=%ctime%_%TIME:~3,2%_%TIME:~6,2%
REM ----------------------------------------------------------------
REM PASS=пароль DB=имясхемы HOST=хост БД LOG=логин БД
REM backuppath=путь_к_бэкапу
REM backupcopypath=путь_к_копии_бэкапа (например, заранее
REM                                     монтированный том)
set PASS=y0ur_p@ssw0rd
set DB=schema_name
set tim=%ctime%
set dat=%DATE%
set ddtim=%dtime%
set HOST=localhost
set PORT=3306
set LOG=root

set backuppath=D:\backup\db\
set backupcopypath=Y:\backup\

REM. > templog.txt


echo MySQL Database Backup v0.1 >> templog.txt
echo -------------------------- >> templog.txt
echo. >> templog.txt

echo HOST:PORT @ USER = %HOST%:%PORT% @ %LOG% >> templog.txt
echo Database tree name = %DB% >> templog.txt
echo NAS mode = %NASMODE% (1 if active) >> templog.txt
echo Backup path = %backuppath% >> templog.txt
echo NAS Backup path = %backupcopypath% >> templog.txt
echo. >> templog.txt
echo %DATE% %TIME%: Backup Started >> templog.txt

set ERRORLEVEL=0
MD %backuppath% >> templog.txt
echo %DATE% %TIME%: Creating backup directory: %ERRORLEVEL% (0 if done) >> templog.txt
echo %DATE% %TIME%: Starting mysqldump >> templog.txt
echo. >> templog.txt

"C:\Program Files\MySQL\MySQL Server 5.6\bin\mysqldump.exe" -v --debug-info=TRUE --log-error=templog.txt ^
        --default-character-set=utf8 --host=%HOST% --port=%PORT% --user %LOG% --password=%PASS% %DB% > %backuppath%mysql_backup__%DB%__%dat%_%tim%.sql
echo ------------------------------------------------------------------------------ >> templog.txt
copy templog.txt %backuppath%mysql_bkplog__%DB%__%dat%_%tim%.txt

echo %DATE% %TIME%: Creating 7zip archive >> templog.txt
7za.exe a -t7z %backuppath%mysql_backup__%DB%__%dat%_%tim%.7z %backuppath%mysql_backup__%DB%__%dat%_%tim%.sql ^
        >> templog.txt
echo ------------------------------------------------------------------------------ >> templog.txt
7za.exe l %backuppath%mysql_backup__%DB%__%dat%_%tim%.7z >> templog.txt
echo ------------------------------------------------------------------------------ >> templog.txt
7za.exe t %backuppath%mysql_backup__%DB%__%dat%_%tim%.7z *.* >> templog.txt
echo ------------------------------------------------------------------------------ >> templog.txt
set ERRORLEVEL=0

echo. >> templog.txt
del %backuppath%mysql_backup__%DB%__%dat%_%tim%.sql >> templog.txt
echo %DATE% %TIME% Deleting unarchived database file ERRORLEVEL is %ERRORLEVEL% (0 if done) >> templog.txt
echo. >> templog.txt
set ERRORLEVEL=0

echo %DATE% %TIME%: Making reserve copy... >> templog.txt
echo %DATE% %TIME%: Copying: %backuppath%mysql_bkplog__%DB%__%dat%_%tim%.txt TO: %backupcopypath%backuplog.txt ^
        >> templog.txt
copy %backuppath%mysql_bkplog__%DB%__%dat%_%tim%.txt %backupcopypath%backuplog.txt  >> templog.txt
echo %DATE% %TIME%: Copying: %backuppath%mysql_backup__%DB%__%dat%_%tim%.7z TO: %backupcopypath%mysql_backup.7z ^
        >> templog.txt
copy %backuppath%mysql_backup__%DB%__%dat%_%tim%.7z %backupcopypath%mysql_backup.7z >> templog.txt
echo %DATE% %TIME%: Making reserve copy ERRORLEVEL is %ERRORLEVEL% (0 if done) >> templog.txt
echo. >> templog.txt
set ERRORLEVEL=0

copy templog.txt %backuppath%mysql_bkplog__%DB%__%dat%_%tim%.txt
copy templog.txt %backupcopypath%backuplog.txt

echo %DATE% %TIME%: Attempting to create log file: %ERRORLEVEL% (0 if done) >> templog.txt
set ERRORLEVEL=0

echo ------------------------------------------------------------------------------ >> templog.txt
echo. >> templog.txt
echo %DATE% %TIME%: Backup successfully DONE. >> templog.txt
copy templog.txt %backuppath%mysql_bkplog__%DB%__%dat%_%tim%.txt
copy templog.txt %backupcopypath%backuplog.txt
del templog.txt
```
Результатом выполнения этого .bat файла будут две запакованные копии дампа MySQL и лог создания с датой в имени файла. Обязательно сверьте версию и место расположение фашего файла: `C:\Program Files\MySQL\MySQL Server 5.6\bin\mysqldump.exe`

Следущий пример запакует расшаренную по сети папку (например, какие-нибудь файлы):
```bat
REM. > templog1s.txt

set ctime=%TIME:~0,2%
if "%ctime:~0,1%" == " " (set ctime=0%ctime:~1,1%) 
set ctime=%ctime%_%TIME:~3,2%_%TIME:~6,2%
set tim=%ctime%
set dat=%DATE%
set ddtim=%dtime%

set backuppath=D:\backup\some\path
set drivnetwork=\\192.168.1.3\somefolder
set useracc=user
set userpass=password

echo Network Shared files backup >> templog1s.txt
echo -------------------------- >> templog1s.txt
echo. >> templog1s.txt

echo Mounting network drive from %drivnetwork% >> templog1s.txt
net use Y: %drivnetwork% /USER:%useracc% %userpass% >> templog1s.txt
7za.exe a -t7z -mx1 -r -ssw -ms=on -mhe %backuppath%%dat%_%tim%.7z Y:\ >> templog1s.txt
echo Removing network drive...
net use Y: /del /yes >> templog1s.txt

echo backup complete, OK >> templog1s.txt
copy templog1s.txt D:\backup\1s\log_%dat%_%tim%.txt >> templog1s.txt
```
Папка backuppath должна быть доступна для чтения для пользователя под которым запускаем этот bat.

При желании, можно так же создать bat для очистки бэкапов и добавить его в планировщик (task scheduler), указав нужный интервал времени между запусками (раз в день/месяц/год). Не забудьте в настройках задания (create new task) выбрать "выполнять даже когда пользователь не вошел в систему". Пути указываете свои.
```bat
REM. > templognas.txt

set ctime=%TIME:~0,2%
if "%ctime:~0,1%" == " " (set ctime=0%ctime:~1,1%) 
set ctime=%ctime%_%TIME:~3,2%_%TIME:~6,2%
set tim=%ctime%
set dat=%DATE%
set ddtim=%dtime%

set backuppath=\\NAS\BackupContainer
set drivnetwork=\\NAS\BackupContainer
set useracc=user
set userpass=password

echo Backup to samba network folder >> templognas.txt
echo -------------------------- >> templognas.txt
echo. >> templognas.txt

echo Mounting network drive from %drivnetwork% >> templognas.txt
net use Y: %drivnetwork% /USER:%useracc% %userpass% >> templognas.txt
copy D:\backup\1s\ Y:\1s\ >> templognas.txt
copy D:\backup\db\ Y:\db\ >> templognas.txt
echo Removing network drive...
net use Y: /del /yes >> templognas.txt

echo backup complete, OK >> templognas.txt
copy templognas.txt D:\backup\1s\log_%dat%_%tim%.txt >> templognas.txt
```
