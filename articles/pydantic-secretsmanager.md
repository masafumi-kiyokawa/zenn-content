---
title: "PydanticでAWSSecretsManagerからシークレットを取得するデモ"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Python", "Pydantic", "SecretsManager"]
published: true
---

Pydanticには環境情報などを管理するツールがあると知った

- [pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)

この中にAWSSecretsManagerからシークレットを取得・管理できるSettingsがある模様
.envから取得するSettingsと組み合わせて使ってみては？という提案も込めて、記事を書く

# 嬉しいこと

1. Pydanticを経由することで型安全に設定値を利用できる
2. [SecretStr](https://docs.pydantic.dev/latest/api/types/#pydantic.types.SecretStr)型を使えばlogやtracebackのマスキングも可能

# 前提
- AWSアカウント、ユーザー作成済み
- AWS CLIをインストール、設定済み
- [uv](https://docs.astral.sh/uv/)を使用
- python3.12で試した
- Pydanticは2.11.3を使用
- PCはMac(多分出てくるコマンド類はWindows同じ)

# デモ
## 準備
```zsh
uv init pydantic-demo
cd pydantic-demo
uv add pydantic-settings boto3 mypy-boto3-secretsmanager
uv run main.py
```

以下が出力されると思われる
```
Hello from pydantic-demo!
```

## シークレット作成
AWS Secrets Managerでシークレットを作成する
シークレットは同じキー、違う値でprod用、dev用の2つ作成する

キーと値は適当
シークレットID(シークレットの名前)を控えておく

AWS CLIでシークレットにアクセスするユーザー(ロール)にシークレットにアクセスする権限がなければ付与しておく

## .envからシークレットIDを取得する
- [Settings Management > Dotenv (.env) support](https://docs.pydantic.dev/latest/concepts/pydantic_settings/#dotenv-env-support)

`.env`を作成
```.env
AWS_SECRETSMANAGER_SECRET_NAME={prod用のシークレットID}
```

`.env.local`を作成
```.env
AWS_SECRETSMANAGER_SECRET_NAME={dev用のシークレットID}
```

`config.py`を作成
```python
from pydantic_settings import (
    BaseSettings,
    SettingsConfigDict,
)

class DotEnvSettings(BaseSettings):
    """.envからの設定読み込み"""

    model_config = SettingsConfigDict(env_file=(".env", ".env.local"), env_file_encoding="utf-8")

    aws_secretsmanager_secret_name: str
```

`.env.local`があるなら`.env`よりも優先する設定

動作確認用に`main.py`を修正
```python
from config import DotEnvSettings


def main():
    print(DotEnvSettings().aws_secretsmanager_secret_name)


if __name__ == "__main__":
    main()
```
mainを実行してみる
```
uv run main.py
```
dev用のシークレットIDが出力されたらOK

## シークレットマネージャーからシークレットの値を取得する
- [Settings Management > AWS Secrets Manager](https://docs.pydantic.dev/latest/concepts/pydantic_settings/#aws-secrets-manager)


`config.py`を修正する
```python
"""SecretsMangerから値を取得するやつ"""

from pydantic import SecretStr
from pydantic_settings import (
    AWSSecretsManagerSettingsSource,
    BaseSettings,
    PydanticBaseSettingsSource,
    SettingsConfigDict,
)


class DotEnvSettings(BaseSettings):
    """.envからの設定読み込み"""

    model_config = SettingsConfigDict(env_file=(".env", ".env.local"), env_file_encoding="utf-8")

    aws_secretsmanager_secret_name: str


class AWSSecretsManagerSettings(BaseSettings):
    """SecretsManagerから値取得"""

    SECRET_KEY: str

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: type[BaseSettings],
        init_settings: PydanticBaseSettingsSource,
        env_settings: PydanticBaseSettingsSource,
        dotenv_settings: PydanticBaseSettingsSource,
        file_secret_settings: PydanticBaseSettingsSource,
    ) -> tuple[PydanticBaseSettingsSource, ...]:
        aws_secrets_manager_settings = AWSSecretsManagerSettingsSource(
            settings_cls,
            DotEnvSettings().aws_secretsmanager_secret_name,
        )
        return (
            init_settings,
            env_settings,
            dotenv_settings,
            file_secret_settings,
            aws_secrets_manager_settings,
        )

```

AWSSecretsManagerSettingsSource()に、sectet_idとして.envから取得したシークレットIDを渡している
`SECRET_KEY`は自身で設定したシークレットのキーと完全に一致している必要がある
`SECRET_KEY`の型は後で確認するために通常の文字列型にしているが、最終的には[SecretStr](https://docs.pydantic.dev/latest/api/types/#pydantic.types.SecretStr)にすると良い

動作確認用に`main.py`を修正
```python
from config import AWSSecretsManagerSettings, DotEnvSettings


def main():
    print(DotEnvSettings().aws_secretsmanager_secret_name)
    print(AWSSecretsManagerSettings().SECRET_KEY)


if __name__ == "__main__":
    main()
```

mainを実行し、dev用のシークレットIDとシークレットの値が出力されたらOK
`.env.local`を削除して再度mainを実行すると、prod用の値が出力されるはず

## シークレットの削除
忘れないうちに削除しておくことを推奨

# おまけ
pydantic-settingsを使って`pyproject.toml`から値を取得することもできる
どこで使うのかは謎だが、[tool.some-table] [tool.other-table]のようにテーブルを分けることで用途ごとにグルーピングできるのは良さそうに思った

- [Settings Management > Other settings source > pyproject.toml](https://docs.pydantic.dev/latest/concepts/pydantic_settings/#pyprojecttoml)

`config,py`
```python
# import文省略
class PyprojectTomlSettings(BaseSettings):
    """PyprojectTomlから値取得"""

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: type[BaseSettings],
        init_settings: PydanticBaseSettingsSource,
        env_settings: PydanticBaseSettingsSource,
        dotenv_settings: PydanticBaseSettingsSource,
        file_secret_settings: PydanticBaseSettingsSource,
    ) -> tuple[PydanticBaseSettingsSource, ...]:
        return (PyprojectTomlConfigSettingsSource(settings_cls),)


class SomeTableSettings(PyprojectTomlSettings):
    """some-tableから取得"""

    foo: str

    model_config = SettingsConfigDict(pyproject_toml_table_header=("tool", "some-table"))


class OtherTableSettings(PyprojectTomlSettings):
    """other-tableから取得"""

    bar: str

    model_config = SettingsConfigDict(pyproject_toml_table_header=("tool", "other-table"))
```

`pyproject.toml`
```toml
[tool.some-table]
foo = "this is some-table variable"

[tool.other-table]
bar = "this is other-table variable"
```

jsonやtoml、yaml用のSettingSourceもあるので公式ドキュメントを参照のこと