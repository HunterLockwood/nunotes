
## Where does the Enter event come from ?

```rust
loop {
      match event::read()? {
          Event::Resize(x, y) => {
              latest_resize = Some((x, y));
          }
          enter @ Event::Key(KeyEvent {
              code: KeyCode::Enter,
              modifiers: KeyModifiers::NONE,
              ..
          }) => {
              let enter = ReedlineRawEvent::convert_from(enter);
              if let Some(enter) = enter {
                  crossterm_events.push(enter);
                  // Break early to check if the input is complete and
                  // can be send to the hosting application. If
                  // multiple complete entries are submitted, events
                  // are still in the crossterm queue for us to
                  // process.
                  paste_enter_state = crossterm_events.len() > EVENTS_THRESHOLD;
                  break;
              }
          }
```

The *Enter* event gets pushed into the crossterm_events Vec.

---

```
for event in crossterm_events.drain(..) {
    match (&mut last_edit_commands, self.edit_mode.parse_event(event)) {
        (None, ReedlineEvent::Edit(ec)) => {
            last_edit_commands = Some(ec);
        }
        (None, other_event) => {
            println!("other_event {:?}", other_event);
            reedline_events.push(other_event);
        }
```

This is where the *Enter* event appears on the scene after being pushed in to
the crossterm_events Vec above.

## Notes about the engine's repaint

```rust
fn read_line_helper(&mut self, prompt: &dyn Prompt) -> Result<Signal> {
     self.painter.initialize_prompt_position()?;
     self.hide_hints = false;
     self.repaint(prompt)?;
```

The first *repaint* in the engine code doesn't really do anything except draw
the prompt initially.  The code does not really have to be there.  If its
not there you just don't see the prompt when you hit ENTER.  However, you
will see the prompt once you type your first key.

---

```rust
EventStatus::Handled => {
     if !paste_enter_state {
         self.repaint(prompt)?;
     }
 }
```

The second *repaint* is the CRITICAL repaint that needs to be there.  Its job
is to actually write each letter out to the screen when you type it.  If its
not there you won't see what you typed until after you hit return.

---

```rust
fn submit_buffer(&mut self, prompt: &dyn Prompt) -> io::Result<EventStatus> {
      let buffer = self.editor.get_buffer().to_string();
      self.hide_hints = true;
      // Additional repaint to show the content without hints etc.
      if let Some(transient_prompt) = self.transient_prompt.take() {
          self.repaint(transient_prompt.as_ref())?;
          self.transient_prompt = Some(transient_prompt);
      } else {
          self.repaint(prompt)?;
      }
```

The third set of repaints are also *not needed*.  If you take them out
there is no effect on whats happening especially in the *basic* example.