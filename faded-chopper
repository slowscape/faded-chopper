--
-- FADEDddDD
--
-- Devine Lu Linvega's simple softcut example 
-- slammed together with SAM,
-- plus some unique additions.
--
-- Sample volume visualized through brightness. 
-- Samples copied with a fade before saved
-- so that clicking is less likely.
-- 
-- Samples numbered in a folder.
-- Samples should not be more that ~2.5 minutes.
--
-- key1       - alt
-- key2       - start/stop record
-- alt + key2 - load sample (file browser)
-- key3       - start/stop playback
-- alt + key3 - save (stereo)
--
-- enc2       - loop start (fine)
-- enc3       - loop end (fine)
-- alt + enc2 - loop start (coarse)
-- alt + enc3 - loop end (coarse)
--
-- v1.5 @slowscape


local te = require 'textentry'
local fileselect = require 'fileselect'

local alt = false
local recording = false
local playing = false
local saved_time = 2.0
local current_position = 0
local last_used_folder = ""
local last_saved_name = 0

local viewport = { width = 128, height = 64, center = 64, middle = 32 }
local recorded_length = 10.0
local loop_start = 0
local loop_end = 10.0
local screen_timer = nil

local FADE_TIME   = 0.01   -- fade in/out baked into each saved sample (seconds)

-- ---------------------------------------------------------------
-- helpers
-- ---------------------------------------------------------------

function check(txt)
  if string.len(txt) > 10 then
    return "too long"
  else
    return ("remaining: " .. 10 - string.len(txt))
  end
end


local function scan_last_index(folder_path)
  local highest = 0
  if util.file_exists(folder_path) then
    local files = util.scandir(folder_path)
    if files then
      for _, f in ipairs(files) do
        local n = string.match(f, "^samz%-(%d+)%.wav$")
        if n then
          local idx = tonumber(n)
          if idx > highest then highest = idx end
        end
      end
    end
  end
  return highest
end


local function set_loop_start(v)
  loop_start = util.clamp(v, 0, loop_end - 0.01)
  softcut.loop_start(1, loop_start)
  softcut.loop_start(2, loop_start)
end


local function set_loop_end(v)
  loop_end = util.clamp(v, loop_start + 0.01, recorded_length)
  softcut.loop_end(1, loop_end)
  softcut.loop_end(2, loop_end)
end


local function reset_loop()
  softcut.buffer_clear()
  loop_start = 0
  loop_end = 350
  recorded_length = 10.0
  softcut.loop_start(1, 0)
  softcut.loop_start(2, 0)
  softcut.loop_end(1, 350)
  softcut.loop_end(2, 350)
  softcut.position(1, 0)
  softcut.position(2, 0)
  current_position = 0
end


local function load_sample(file)
  if file == nil or file == "cancel" then return end
  local chan, samples, rate = audio.file_info(file)
  local sample_len = samples / rate
  softcut.buffer_clear()
  if chan >= 2 then
    softcut.buffer_read_stereo(file, 0, 0, -1, 0, 1)
  else
    softcut.buffer_read_mono(file, 0, 0, -1, 1, 1, 0, 1)
    softcut.buffer_read_mono(file, 0, 0, -1, 1, 2, 0, 1)
  end
  recorded_length = sample_len
  loop_start = 0
  loop_end = sample_len
  softcut.loop_start(1, 0)
  softcut.loop_start(2, 0)
  softcut.loop_end(1, sample_len)
  softcut.loop_end(2, sample_len)
  softcut.position(1, 0)
  softcut.position(2, 0)
  playing = true
  softcut.play(1, 1)
  softcut.play(2, 1)
end


-- ---------------------------------------------------------------
-- write buffer (copy with baked fades then save)
-- ---------------------------------------------------------------

local SCRATCH_PAD = 1.0    -- gap between recorded_length and scratch region (seconds)
local TOTAL_BUF   = 350.0  -- total softcut buffer length

function write_buffer(name)
  if name == nil or check(name) == "too long" then return end
  last_used_folder = name
  local folder_path = _path.audio .. "a-samples/" .. last_used_folder

  if util.file_exists(folder_path) == false then
    util.make_dir(folder_path)
  end

  last_saved_name = scan_last_index(folder_path) + 1

  local dur         = loop_end - loop_start
  local scratch_dst = recorded_length + SCRATCH_PAD

  -- safety check: make sure the scratch region fits in the buffer
  if scratch_dst + dur > TOTAL_BUF then
    print("samz: not enough buffer space to bake fades — saving without fade")
    local file_path = folder_path .. "/fc-" .. string.format("%03d", last_saved_name) .. ".wav"
    softcut.buffer_write_stereo(file_path, loop_start, dur)
    saved_time = util.time()
    return
  end

  -- copy selected region to scratch space with fade baked in
  -- buffer_copy_stereo(start_src, start_dst, dur, fade_time, preserve, reverse)
  softcut.buffer_copy_stereo(loop_start, scratch_dst, dur, FADE_TIME, 0, 0)

  clock.run(function()
    clock.sleep(0.1)  -- brief yield so copy command clears the audio engine queue
    local file_path = folder_path .. "/fc-" .. string.format("%03d", last_saved_name) .. ".wav"
    softcut.buffer_write_stereo(file_path, scratch_dst, dur)
    print("Buffer saved as " .. file_path)
    saved_time = util.time()
    softcut.buffer_clear_region(scratch_dst, dur + FADE_TIME, FADE_TIME, 0)
  end)
end


-- ---------------------------------------------------------------
-- phase poll
-- ---------------------------------------------------------------

local function update_positions(voice, position)
  if voice == 1 then
    current_position = position
  end
end


-- ---------------------------------------------------------------
-- init
-- ---------------------------------------------------------------

function init()
  audio.level_cut(1)
  audio.level_adc_cut(1)
  audio.level_eng_cut(1)

  -- Voice 1 - LEFT (buffer 1, ADC L)
  softcut.enable(1, 1)
  softcut.buffer(1, 1)
  softcut.level(1, 1)
  softcut.pan(1, -1)
  softcut.play(1, 0)
  softcut.rate(1, 1)
  softcut.loop_start(1, 0)
  softcut.loop_end(1, 350)
  softcut.loop(1, 1)
  softcut.fade_time(1, FADE_TIME)
  softcut.rec(1, 0)
  softcut.rec_level(1, 1)
  softcut.pre_level(1, 1)
  softcut.recpre_slew_time(1, FADE_TIME)
  softcut.position(1, 0)
  softcut.filter_dry(1, 1)
  softcut.level_input_cut(1, 1, 1.0)

  -- Voice 2 - RIGHT (buffer 2, ADC R)
  softcut.enable(2, 1)
  softcut.buffer(2, 2)
  softcut.level(2, 1)
  softcut.pan(2, 1)
  softcut.play(2, 0)
  softcut.rate(2, 1)
  softcut.loop_start(2, 0)
  softcut.loop_end(2, 350)
  softcut.loop(2, 1)
  softcut.fade_time(2, FADE_TIME)
  softcut.rec(2, 0)
  softcut.rec_level(2, 1)
  softcut.pre_level(2, 1)
  softcut.recpre_slew_time(2, FADE_TIME)
  softcut.position(2, 0)
  softcut.filter_dry(2, 1)
  softcut.level_input_cut(2, 2, 1.0)

  softcut.phase_quant(1, 0.01)
  softcut.event_phase(update_positions)
  softcut.poll_start_phase()

  screen_timer = metro.init()
  screen_timer.time = 1 / 15
  screen_timer.event = function() redraw() end
  screen_timer:start()
end


-- ---------------------------------------------------------------
-- keys
-- ---------------------------------------------------------------

function key(n, z)
  if n == 1 then
    alt = z == 1 and true or false
  end

  if n == 2 and z == 1 then
    if alt then
      local browse_path = _path.audio .. "a-samples/"
      if util.file_exists(browse_path) == false then
        util.make_dir(browse_path)
      end
      screen_timer:stop()
      fileselect.enter(browse_path, function(file)
        screen_timer:start()
        load_sample(file)
      end)
      alt = false
    else
      if recording == false then
        reset_loop()
        softcut.rec(1, 1)
        softcut.rec(2, 1)
        softcut.play(1, 1)
        softcut.play(2, 1)
        recording = true
        playing = false
      else
        softcut.rec_level(1, 0)
        softcut.rec_level(2, 0)
        clock.run(function()
          clock.sleep(0.08)
          softcut.rec(1, 0)
          softcut.rec(2, 0)
          recorded_length = current_position
          loop_start = 0
          loop_end = recorded_length
          softcut.loop_end(1, recorded_length)
          softcut.loop_end(2, recorded_length)
          softcut.position(1, 0)
          softcut.position(2, 0)
          softcut.rec_level(1, 1)
          softcut.rec_level(2, 1)
          recording = false
          playing = true
        end)
      end
    end

  elseif n == 3 and z == 1 then
    if alt then
      te.enter(write_buffer, last_used_folder, 'Folder Name: ')
      alt = false
    else
      if recording then
        -- ignore while recording
      elseif playing then
        softcut.play(1, 0)
        softcut.play(2, 0)
        playing = false
      else
        softcut.position(1, 0)
        softcut.position(2, 0)
        softcut.play(1, 1)
        softcut.play(2, 1)
        playing = true
      end
    end
  end
end


-- ---------------------------------------------------------------
-- encoders
-- ---------------------------------------------------------------

function enc(n, d)
  local step = alt and 0.25 or 0.005
  if n == 2 then
    set_loop_start(loop_start + d * step)
  elseif n == 3 then
    set_loop_end(loop_end + d * step)
  end
end


-- ---------------------------------------------------------------
-- screen
-- ---------------------------------------------------------------

local function draw_frame()
  screen.rect(1, 1, viewport.width - 1, viewport.height - 1)
  screen.stroke()
end


local function time_to_x(t, pad, width, scale)
  return pad + (t / scale) * width
end


function redraw()
  screen.aa(0)
  screen.clear()
  screen.level(15)
  screen.line_width(1)

  draw_frame()

  local pad    = 8
  local width  = viewport.width - (2 * pad)
  local bar_y  = viewport.middle + 6
  local bar_scale = recording
    and math.max(current_position, 0.1)
    or  recorded_length

  local x_left  = pad
  local x_right = pad + width
  local x_start = time_to_x(loop_start, pad, width, bar_scale)
  local x_end   = recording
    and x_right
    or  time_to_x(loop_end, pad, width, bar_scale)
  local x_pos   = time_to_x(
    util.clamp(current_position, 0, bar_scale),
    pad, width, bar_scale
  )

  -- dim background (full recorded length)
  screen.level(2)
  screen.move(x_left, bar_y)
  screen.line(x_right, bar_y)
  screen.stroke()

  -- active region between in/out points
  screen.level(recording and 10 or (playing and 15 or 5))
  screen.move(x_start, bar_y)
  screen.line(x_end, bar_y)
  screen.stroke()

  -- playhead tick
  if playing or recording then
    screen.level(15)
    screen.move(x_pos, bar_y - 3)
    screen.line(x_pos, bar_y + 3)
    screen.stroke()
  end

  -- outer boundary nubs (total file edges, dim)
  screen.level(4)
  screen.move(x_left,  bar_y - 2)
  screen.line(x_left,  bar_y + 2)
  screen.stroke()
  screen.move(x_right, bar_y - 2)
  screen.line(x_right, bar_y + 2)
  screen.stroke()

  -- in/out point nubs (brighter, moveable)
  screen.level(12)
  screen.move(x_start, bar_y - 4)
  screen.line(x_start, bar_y + 4)
  screen.stroke()
  screen.move(x_end,   bar_y - 4)
  screen.line(x_end,   bar_y + 4)
  screen.stroke()

  -- state label
  screen.level(recording and 15 or 4)
  screen.move(viewport.center, 10)
  if recording then
    screen.text_center("recording")
  elseif playing then
    screen.text_center("playing")
  else
    screen.text_center("stopped")
  end

  -- in/out time readouts
  screen.level(15)
  screen.move(pad + 2, bar_y + 12)
  screen.text(string.format("%.2f", loop_start))
  screen.move(viewport.width - pad - 2, bar_y + 12)
  screen.text_right(string.format("%.2f", loop_end))

  -- current position
  screen.level(8)
  screen.move(viewport.center, bar_y + 12)
  screen.text_center(string.format("%.2f", current_position))

  -- key hints
  screen.level(4)
  screen.move(pad, viewport.height - 3)
  if recording then
    screen.text("stop")
  elseif alt then
    screen.text("load")
  else
    screen.text("rec")
  end

  screen.move(viewport.width - pad, viewport.height - 3)
  if alt then
    screen.text_right("save")
  elseif recording then
    screen.text_right(" - ")
  elseif playing then
    screen.text_right("stop")
  else
    screen.text_right("play")
  end

  -- saved flash
  if util.time() - saved_time <= 1.5 then
    screen.level(4)
    screen.move(viewport.center, bar_y - 10)
    screen.text_center("saved samz-" .. string.format("%03d", last_saved_name) .. ".wav")
  end

  screen.update()
end
