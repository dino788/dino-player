import os
import re
import shutil
import subprocess
from PIL import Image, ImageDraw, ImageFont

APK_INPUT = "Ibo Pro Player(@Monxx_Mod).apk"
SRC_DIR = "dinho_src"
APK_UNSIGNED = "dinho_unsigned.apk"
LOGO_FILE = "dino_vitan_logo.png"


def run(cmd):
  print(f"[*] Running: {cmd}")
  subprocess.run(cmd, shell=True, check=True)


def generate_embedded_logo(output_path):
  # Kay-creyi l-logo "dino vitan" b l-khder automatiquement
  size = 512
  img = Image.new("RGBA", (size, size), (22, 29, 34, 255))
  draw = ImageDraw.Draw(img)

  # Circle green emblem
  draw.ellipse((116, 60, 396, 340), outline=(0, 230, 118, 255), width=18)
  # Play triangle inside
  draw.polygon([(235, 165), (235, 235), (300, 200)], fill=(118, 255, 3, 255))

  # Text "dino vitan"
  try:
    font = ImageFont.truetype(
        "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf", 52
    )
  except:
    font = ImageFont.load_default()

  draw.text((105, 375), "dino", fill=(0, 230, 118, 255), font=font)
  draw.text((245, 375), "vitan", fill=(255, 255, 255, 255), font=font)

  img.save(output_path, "PNG")
  print(f"[+] Logo '{output_path}' m-integri w t-creya b najah!")


def patch_app():
  if not os.path.exists(APK_INPUT):
    raise FileNotFoundError(f"Malqitch l-fichier: {APK_INPUT}")

  # 0. Bni l-logo direct mn l-code
  generate_embedded_logo(LOGO_FILE)

  # 1. Decompile APK
  if os.path.exists(SRC_DIR):
    shutil.rmtree(SRC_DIR)
  run(f'java -jar apktool.jar d "{APK_INPUT}" -o {SRC_DIR} -f')

  # 2. Tbdil Smiya l "Dinho Player"
  strings_path = os.path.join(SRC_DIR, "res", "values", "strings.xml")
  if os.path.exists(strings_path):
    with open(strings_path, "r", encoding="utf-8", errors="ignore") as f:
      content = f.read()
    content = re.sub(
        r'(<string name="app_name">).*?(</string>)',
        r"\1Dinho Player\2",
        content,
    )
    content = re.sub(r"Ibo Pro Player|IBO Player", "Dinho Player", content)
    with open(strings_path, "w", encoding="utf-8") as f:
      f.write(content)

  # 3. Tbdil Loun l-Khder (Green Theme)
  colors_path = os.path.join(SRC_DIR, "res", "values", "colors.xml")
  if os.path.exists(colors_path):
    with open(colors_path, "r", encoding="utf-8", errors="ignore") as f:
      c_data = f.read()
    c_data = re.sub(
        r'(<color name="colorPrimary">).*?(</color>)', r"\1#1B5E20\2", c_data
    )
    c_data = re.sub(
        r'(<color name="colorPrimaryDark">).*?(</color>)',
        r"\1#0D3B12\2",
        c_data,
    )
    c_data = re.sub(
        r'(<color name="colorAccent">).*?(</color>)', r"\1#00E676\2", c_data
    )
    with open(colors_path, "w", encoding="utf-8") as f:
      f.write(c_data)

  # 4. Replace Logo & Icons f ga3 les dossiers dyal res
  for root, dirs, files in os.walk(os.path.join(SRC_DIR, "res")):
    for file in files:
      if file in [
          "ic_launcher.png",
          "ic_launcher_round.png",
          "logo.png",
          "app_logo.png",
      ]:
        shutil.copy(LOGO_FILE, os.path.join(root, file))

  # 5. Smali Patch: Bypass MAC/Key + Persistence Fix
  for root, dirs, files in os.walk(SRC_DIR):
    if "smali" not in root:
      continue
    for file in files:
      if not file.endswith(".smali"):
        continue
      fpath = os.path.join(root, file)
      with open(fpath, "r", encoding="utf-8", errors="ignore") as f:
        s_code = f.read()

      modified = False

      if (
          "isActivated" in s_code
          or "checkActivation" in s_code
          or "isExpired" in s_code
      ):
        s_code = re.sub(
            r"(\.method public .*?isActivated\(\)Z\s+\.locals \d+)",
            r"\1\n    const/4 v0, 0x1\n    return v0",
            s_code,
        )
        s_code = re.sub(
            r"(\.method public .*?isExpired\(\)Z\s+\.locals \d+)",
            r"\1\n    const/4 v0, 0x0\n    return v0",
            s_code,
        )
        modified = True

      if "SharedPreferences$Editor;->clear()" in s_code:
        s_code = s_code.replace(
            "invoke-interface {v0},"
            " Landroid/content/SharedPreferences$Editor;->clear()Landroid/content/SharedPreferences$Editor;",
            "# keep server saved",
        )
        s_code = s_code.replace(
            "invoke-interface {v1},"
            " Landroid/content/SharedPreferences$Editor;->clear()Landroid/content/SharedPreferences$Editor;",
            "# keep server saved",
        )
        modified = True

      if modified:
        with open(fpath, "w", encoding="utf-8") as f:
          f.write(s_code)

  # 6. Rebuild b --use-aapt2 bach n-tfadaw Build Failed
  run(
      f"java -jar apktool.jar b {SRC_DIR} -o {APK_UNSIGNED} --use-aapt2"
      " --copy-original"
  )
  print("[✓] Build njah bla errors!")


if __name__ == "__main__":
  patch_app()
