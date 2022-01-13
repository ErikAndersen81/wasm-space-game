use std::f32::consts::PI;

use bevy::{core::FixedTimestep, prelude::*};
use rand::{thread_rng, Rng};

use crate::{
    ActiveEnemies, Enemy, FromEnemy, Laser, Materials, Speed, WinSize, MAX_ENEMIES,
    MAX_FORMATION_MEMBERS, TIME_STEP,
};

pub struct EnemyPlugin;

impl Plugin for EnemyPlugin {
    fn build(&self, app: &mut AppBuilder) {
        app.insert_resource(FormationMaker::default())
            .add_system(enemy_laser_movement.system())
            .add_system(enemy_movement.system())
            .add_system_set(
                SystemSet::new()
                    .with_run_criteria(FixedTimestep::step(1.0))
                    .with_system(enemy_spawn.system()),
            )
            .add_system_set(
                SystemSet::new()
                    .with_run_criteria(FixedTimestep::step(0.9))
                    .with_system(enemy_fire.system()),
            );
    }
}

fn enemy_spawn(
    mut commands: Commands,
    mut active_enemies: ResMut<ActiveEnemies>,
    mut formation_maker: ResMut<FormationMaker>,
    win_size: Res<WinSize>,
    materials: Res<Materials>,
) {
    if active_enemies.0 < MAX_ENEMIES {
        let formation = formation_maker.make(&win_size);
        let (x, y) = formation.start;

        commands
            .spawn_bundle(SpriteBundle {
                material: materials.enemy.clone(),
                transform: Transform {
                    translation: Vec3::new(x, y, 10.0),
                    scale: Vec3::new(0.3, 0.3, 1.0),
                    ..Default::default()
                },
                ..Default::default()
            })
            .insert(Enemy)
            .insert(Speed::default())
            .insert(formation);

        active_enemies.0 += 1;
    }
}

fn enemy_fire(
    mut commands: Commands,
    materials: Res<Materials>,
    enemy_query: Query<&Transform, With<Enemy>>,
) {
    for &tf in enemy_query.iter() {
        let x = tf.translation.x;
        let y = tf.translation.y;
        commands
            .spawn_bundle(SpriteBundle {
                material: materials.enemy_laser.clone(),
                transform: Transform {
                    translation: Vec3::new(x, y - 15., 0.),
                    scale: Vec3::new(0.2, 0.2, 1.),
                    ..Default::default()
                },
                ..Default::default()
            })
            .insert(Laser)
            .insert(FromEnemy)
            .insert(Speed::default());
    }
}

fn enemy_laser_movement(
    mut commands: Commands,
    win_size: Res<WinSize>,
    mut laser_query: Query<(Entity, &Speed, &mut Transform), (With<Laser>, With<FromEnemy>)>,
) {
    for (entity, speed, mut tf) in laser_query.iter_mut() {
        tf.translation.y -= speed.0 * TIME_STEP;
        if tf.translation.y < -win_size.h / 2. - 50. {
            commands.entity(entity).despawn();
        }
    }
}

fn enemy_movement(mut query: Query<(&mut Transform, &Speed, &mut Formation), With<Enemy>>) {
    for (mut tf, speed, mut formation) in query.iter_mut() {
        let max_distance = TIME_STEP * speed.0;
        let x_org = tf.translation.x;
        let y_org = tf.translation.y;
        let (x_offset, y_offset) = formation.offset;
        let (x_radius, y_radius) = formation.radius;

        let dir = if formation.start.0 > 0. { 1. } else { -1. };
        let angle =
            formation.angle + dir * speed.0 * TIME_STEP / (x_radius.min(y_radius) * PI / 2.);

        let x_dst = x_radius * angle.cos() + x_offset;
        let y_dst = y_radius * angle.sin() + y_offset;

        let dx = x_org - x_dst;
        let dy = y_org - y_dst;
        let distance = (dx * dx + dy * dy).sqrt();
        let distance_ratio = if distance == 0.0 {
            0.
        } else {
            max_distance / distance
        };

        let x = x_org - dx * distance_ratio;
        let x = if dx > 0. { x.max(x_dst) } else { x.min(x_dst) };
        let y = y_org - dy * distance_ratio;
        let y = if dy > 0. { y.max(y_dst) } else { y.min(y_dst) };

        if distance < max_distance * speed.0 / 20. {
            formation.angle = angle;
        }

        tf.translation.x = x;
        tf.translation.y = y;
    }
}

// Formation
#[derive(Default, Clone)]
struct Formation {
    start: (f32, f32),
    radius: (f32, f32),
    offset: (f32, f32),
    angle: f32,
    #[allow(unused)]
    group_id: u32,
}

#[derive(Default)]
struct FormationMaker {
    group_seq: u32,
    current_formation: Option<Formation>,
    current_formation_members: u32,
}

impl FormationMaker {
    fn make(&mut self, win_size: &WinSize) -> Formation {
        match (
            &self.current_formation,
            self.current_formation_members >= MAX_FORMATION_MEMBERS,
        ) {
            (None, _) | (_, true) => {
                let mut rng = thread_rng();
                let h_span = win_size.h / 2. - 100.;
                let w_span = win_size.w / 4.;
                let x = if rng.gen::<bool>() {
                    win_size.w
                } else {
                    -win_size.w
                };
                let y = rng.gen_range(-h_span..h_span) as f32;
                let start = (x, y);

                let offset = (rng.gen_range(-w_span..w_span), rng.gen_range(0.0..h_span));
                let radius = (rng.gen_range(80.0..150.), 100.);
                let angle = (y - offset.0).atan2(x - offset.1);

                self.group_seq += 1;
                let group_id = self.group_seq;
                let formation = Formation {
                    start,
                    offset,
                    radius,
                    angle,
                    group_id,
                };

                self.current_formation = Some(formation.clone());
                self.current_formation_members = 1;
                formation
            }
            (Some(tmpl), false) => {
                self.current_formation_members += 1;
                tmpl.clone()
            }
        }
    }
}
