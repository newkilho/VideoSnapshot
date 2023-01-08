program VideoSnapshot;

{$APPTYPE CONSOLE}

{$R *.res}

uses
  Winapi.Windows, System.SysUtils, K.Strings;

procedure Die(Str: string);
begin
  WriteLn(Str);
  Halt;
end;

function GetDuration(Str: string): Integer;
var
  Item: string;
begin
  Item := Parsing(Str, 'Duration: ', ',');
  if Length(Item) = 11 then
    Result := StrToInt(Copy(Item, 1, 2))*60*60+
              StrToInt(Copy(Item, 4, 2))*60+
              StrToInt(Copy(Item, 7, 2))
  else
    Result := -1;
end;

function GetDosOutput(CommandLine: string; Work: string = ''): string;
var
  SA: TSecurityAttributes;
  SI: TStartupInfo;
  PI: TProcessInformation;
  StdOutPipeRead, StdOutPipeWrite: THandle;
  WasOK: Boolean;
  Buffer: array[0..255] of AnsiChar;
  BytesRead: Cardinal;
  WorkDir: string;
  Handle: Boolean;
  Data: AnsiString;
begin
  if Work = '' then Work := ExtractFilePath(ParamStr(0));
  Data := '';

  with SA do begin
    nLength := SizeOf(SA);
    bInheritHandle := True;
    lpSecurityDescriptor := nil;
  end;
  CreatePipe(StdOutPipeRead, StdOutPipeWrite, @SA, 0);
  try
    with SI do
    begin
      FillChar(SI, SizeOf(SI), 0);
      cb := SizeOf(SI);
      dwFlags := STARTF_USESHOWWINDOW or STARTF_USESTDHANDLES;
      wShowWindow := SW_HIDE;
      hStdInput := GetStdHandle(STD_INPUT_HANDLE); // don't redirect stdin
      hStdOutput := StdOutPipeWrite;
      hStdError := StdOutPipeWrite;
    end;
    WorkDir := Work;
    Handle := CreateProcess(nil, PChar(CommandLine),
                            nil, nil, True, 0, nil,
                            PChar(WorkDir), SI, PI);
    CloseHandle(StdOutPipeWrite);
    if Handle then
      try
        repeat
          WasOK := ReadFile(StdOutPipeRead, Buffer, SizeOf(Buffer)-1, BytesRead, nil);
          if BytesRead > 0 then
          begin
            Buffer[BytesRead] := #0;
            Data := Data + AnsiString(Buffer);
          end;
        until not WasOK or (BytesRead = 0);
        WaitForSingleObject(PI.hProcess, INFINITE);
      finally
        CloseHandle(PI.hThread);
        CloseHandle(PI.hProcess);
      end;
  finally
    CloseHandle(StdOutPipeRead);
  end;

  Result := UTF8ToString(Data);
end;

function GetParam(var Loop: Integer): string;
begin
  Inc(Loop);
  if Loop <= ParamCount then Result := ParamStr(Loop) else Die('Syntax error in parameters or arguments.');
end;

var
  PathName, FFMpeg, Source, Output: string;
  Count, Start, Duration, Loop: Integer;
begin
  // Start
  WriteLn('Video Snapshot version 0.9');
  WriteLn('Copyright 2019. Kilho.net. All rights reserved.');
  WriteLn('');

  // Init
  PathName := ExtractFilePath(ParamStr(0));
  FFMpeg := PathName + 'vendor\ffmpeg\ffmpeg.exe';
  Start := 20;
  Count := 1;

  if not FileExists(FFMpeg) then Die('FFMpeg.exe is Not Found.');

  Loop := 0;
  while Loop <= ParamCount do
  begin
    if ParamStr(Loop) = '/i' then Source := GetParam(Loop);
    if ParamStr(Loop) = '/o' then PathName := GetParam(Loop);
    if ParamStr(Loop) = '/s' then Count := StrToIntDef(GetParam(Loop), 1);
    if ParamStr(Loop) = '/n' then Count := StrToIntDef(GetParam(Loop), 1);

    Inc(Loop);
  end;

  if Source = '' then
  begin
    WriteLn('Usage: VideoSnapshot /i <value> [/o <value>] [/n <value>]');
    WriteLn('');
    WriteLn('/i <value>       source video filename');
    WriteLn('/o <value>       output path of the snapshots');
    WriteLn('/s <value>       start time of snapshots to save (default: 20)');
    WriteLn('/n <value>       count of snapshots to save (default: 1)');

    Halt;
  end;

  if not FileExists(Source) then Die(ExtractFileName(Source)+' is not found.');

  // Snapshot
  Duration := GetDuration(GetDosOutput('"'+FFMpeg+'" -i "'+Source+'"'));
  if Duration < Round(Duration/Count)*(Count-1)+Start then Die('Video is too short.');

  for Loop := 1 to Count do
  begin
    Output := PathName+Format('snapshot_%0.3d.jpg', [Loop]);
    if FileExists(Output) then DeleteFile(Output);

    WriteLn(' #'+Loop.ToString+': '+Output);
    GetDosOutput('"'+FFMpeg+'" -accurate_seek -ss '+FormatDateTime('hh:nn:ss.000', (Round(Duration/Count)*(Loop-1)+Start)/SecsPerDay)+' -i "'+Source+'" -frames:v 1 "'+Output+'"');
  end;
end.
